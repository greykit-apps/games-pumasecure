# PumaSecure — Bug & Balance Review (2026-07-03)

> **Status (2026-09-07) — second pass, branch `fix/review-2026-09`:** a full re-read plus a headless simulation harness (the real script driven in Node with bot players, 30 seeds per preset) found the remaining P1/P2 items still open and several new ones. All fixed in this pass:
> - **Early win**: the lockout condition now requires a secured network (every discovered node scanned, no unaddressed medium+ vulns, eroded mitigations don't count) and a full 60s lockout — it used to fire at exactly 5:00 on Easy/Tutorial with 60-100 unfixed vulns. Onsite operators now leave after four attempts (`onsite` was never cleared, which made the lockout win impossible once anyone was dispatched).
> - **Physical intrusion** now respects defenses (hardened nodes are never targeted; EDR ×0.6, fully patched ×0.5) and counts *mitigated* physical vulns — the category is mitigate-only, so the old `fixed` checks could never be true.
> - **Head start**: the red team now probes during it (the probe branch was unreachable).
> - **Persistence race**: Tier 2 alert delay 3-6s (was 5-10), anomaly investigation 4-8s (was 5-10), persistence 20-40s (was 15-30), IR 8-14s on EDR nodes. Failed attacks on Tier 2 nodes now raise benign anomalies ~35% of the time (item 15). Re-entry waits 20s after IR and takes 6-10s (was 8s / 4-7s).
> - **Scoring**: response is per incident against a 180s window (stale `reclaimedAt` made 28 of 30 Normal runs count negative response times — item 11); efficiency is `100·√(riskRatio·min(1, riskRatio/spendRatio))` so hoarding can't score 100.
> - **Bugs**: game loop kept running after `endRun` (budget drips on the results screen) and shortcuts acted on dead runs (item 12); budget could go negative via the rework multiplier (item 8); a free playbook fix was consumed when all analysts were busy; strategic buffs re-applied on doubling and re-entry (item 13); doubled helpers now stand down when the primary completes; incarceration no longer splices mid-iteration (item 14); persistence was visible in the detail panel, map and asset list on silently compromised nodes (now gated on `persistenceKnown`); per-vuln `cost/time/successRate` were never read (supply-chain now uses `fixOverrides`); the intel filter was inverted (no intel meant no filtering); duplicate `class` attributes (item 9); `_lastHintCheck` (item 10); Investigate button label; sounds before validation; hire while paused; dead code (item 25); Zero Day modifier vuln is mitigate-only instead of unfixable; Strong IAM now also suppresses default-credential vulns; achievements (item 27).
> - **Presets** retuned from simulation: Normal 1 pentester (4.3), Hard 2 pentesters at 1.0× with 3 analysts (6.4), Nightmare 4 pentesters with 2 analysts (10.0). Measured with the heuristic bot on Enterprise: Tutorial A/B, Easy B, Normal C/B with ~30% critical breach, Hard mostly F under the auto-F gate, Nightmare all F.
> - **Still open**: pinch-zoom on touch (item 28), queuing actions while paused (item 23). The simulation harness lives outside the repo for now.

> **Status (2026-07-03):**
> - **P0 items 1–7: all fixed** (Configure exploitability, physical category ID, fake vuln, early win via lockedOutTimer, rebuilt-node cleanliness, Zero Day modifier implemented, insider persistence).
> - **Balance pass complete — items 16–22 and 24 fixed**: efficiency axis reworked (ratio-based, ×70 scale), S-grade + auto-F gates unified at difficulty 6.0, intel anchor now derived dynamically from the Normal preset (currently 4.7), F runs earn 25% consolation intel, mitigation erosion implemented (60s grace → decay to 35% over 4 min, Re-Mitigate to refresh), Supply Chain targets all discovered nodes at $4–8K fix cost, persistence re-entry implemented (4–7s backdoor re-entry bypassing exploits/hardening), physical risk classified by node type (IoT/Wi-Fi/workstations low-risk, cloud exempt), template unlock curve slowed (40→850).
> - **Bonus fix found during balance testing**: Mitigate was only offered on 3 of 15 vuln categories, so Governance-locked vulns in the other 12 had *zero* fix buttons. Mitigate is now universally available (it erodes, so it's balanced).
> - **Still open**: P1 items 8–15 (except 16), P2 item 23 (pause queuing), P3 items 25–28.
> Line numbers below refer to commit e30f23e and have drifted after the fixes above.

Full-code review of `index.html` (~6,000 lines) against README.md, GAME_THEORY.md, and the in-game help text. Each item is written to be usable as a standalone fix prompt. Line numbers refer to the current `index.html` on `main` (commit e30f23e).

---

## P0 — Bugs that break documented core mechanics

### 1. Configure has zero defensive effect (severity is never used in exploit rolls)
**Where:** `rtTryExploit()` at index.html:4063–4139, Configure effect at index.html:3477–3484, help text at index.html:887 and :961.
**Problem:** The red team's exploit chance is computed purely from `vuln.exploitability` (adjusted by attack vector, mitigation factor, buffs, hardening, EDR). `vuln.severity` is never consulted anywhere in the attack path. The Configure fix only reduces `severity` (by 3–5 points), so Configure improves your posture *score* but does absolutely nothing to stop the red team. The in-game help explicitly claims the opposite: "A cheap Configure can downgrade a Critical to Medium, making it far less exploitable."
**Fix prompt:** Make Configure defensively meaningful. Either (a) have Configure also reduce `exploitability` proportionally to the severity reduction, or (b) factor severity into `rtTryExploit`'s per-vuln chance (e.g. `chance *= 0.4 + 0.06 * severity`) so that severity reduction matters. Option (b) also makes the "severity" stat honest everywhere. Update tooltips/help text to match whichever behavior you pick.

### 2. `physical_security` category ID doesn't exist — the whole physical-vuln loop is dead
**Where:** index.html:4344 (`ptPickPhysicalTarget`), :4485 and :4508 (physical penetration resolution), :4495 (fake vuln). The actual category ID is `physical` (defined at index.html:1158).
**Problem:** Four call sites check `v.category.id === 'physical_security'`, but the category is `physical`. Consequences:
- High-value (zone 3+) nodes with unfixed physical vulns are **never** selected as physical penetration targets (the documented "server room USB-jack" scenario can't happen via vulns).
- The `physVulnBonus` to intrusion success is always 0.
- `physSecFixed` is always false, so **fixing physical security vulns never raises the capture chance** — the documented player incentive to fix physical vulns is entirely non-functional.
**Fix prompt:** Replace all four `'physical_security'` comparisons with `'physical'`, and give the fake vuln at :4495 the real category object (`VULN_CATEGORIES.find(c => c.id === 'physical')`) plus the fields the detail panel expects (see item 3).

### 3. Physical-breach fake vuln renders broken and permanently poisons the score
**Where:** index.html:4495–4497, rendered by `renderDetail()` at index.html:5406–5426, scored in `endRun()` at index.html:4871–4880.
**Problem:** On a successful physical breach, a synthetic vuln `{ name, category: {id:'physical_security'}, severity: 10, exploitability: 1, revealed: true }` is pushed into `node.vulns`. It lacks `attackVector`, `category.name`, and `category.fixTypes`, so the detail panel shows "undefined · undefined" and offers no fix buttons — it is an unfixable, revealed, severity-10 vuln that permanently drags the Posture score (25% weight) and Assessment denominator for the rest of the run. It also isn't counted in `R.totalVulns`.
**Fix prompt:** Either don't add a vuln at all (just set compromise + persistence), or add a well-formed vuln: full category object, `attackVector: 'Physical'`, `fixTypes` that allow mitigation, and increment `R.totalVulns`. Decide whether it should count against Posture; if it represents the intrusion event rather than a weakness, exclude it from scoring.

### 4. Early win is effectively unreachable
**Where:** gameLoop early-win check at index.html:3100–3107; pentester idle handling at index.html:4537–4560.
**Problem:** The early-win condition requires `pentesters.every(p => p.state === 'idle')`. But an idle pentester re-targets within its very next update tick (idle → scanning immediately, since `ptPickTarget` considers any non-compromised node "viable" even if fully patched). Pentesters are 'idle' for roughly one frame each, so with 1–5 pentesters cycling scan(5–12s)→exploit(4–10s)→cooldown(3–5s), the all-idle condition essentially never holds. The documented early-win ("secure all discovered nodes, eliminate compromises, no footholds after 5 minutes") never fires. This also makes the **Speed Run achievement unobtainable** (index.html:4982 — requires `elapsed < 480`, only possible via early win).
Additionally: if **all** pentesters are incarcerated, `pentesters.length > 0` fails, so wiping out the entire red team also blocks the early win — the game just runs out the clock.
**Fix prompt:** Rewrite the early-win condition to not depend on transient pentester state. Suggested: after 300s, end early when (a) no compromised nodes, (b) no footholds, (c) all discovered nodes scanned, and (d) either all pentesters have failed N consecutive exploit attempts (track a counter) or `rt.stuckTimer` exceeds a threshold, or all pentesters are incarcerated (`pentesters.length === 0` should be an immediate win). Consider logging progress toward the early win so players know it exists.

### 5. Burn & Rebuild's "clean system" regenerates vulnerabilities when re-scanned
**Where:** `processRebuilds()` at index.html:3629–3687 (sets `vulns = []`, `scanned = false`); `processScans()` completion at index.html:3280–3284 (regenerates vulns when `vulns.length === 0`).
**Problem:** Docs promise "the node comes back clean with zero vulnerabilities." But rebuild resets `scanned` to false, and any later vuln scan sees an empty `vulns` array and calls `generateVulns()` — conjuring a fresh set of vulnerabilities on the supposedly clean box. The red team's `rtTryExploit`/recon paths (:4066, :4320, :4571) do the same. Also, because the node is unscanned, it drops to Detection Tier 3 (blind) and blocks the early-win "all discovered nodes scanned" condition.
**Fix prompt:** Add a `node.rebuilt = true` flag (or keep `scanned = true, deepScanned = false` after rebuild) so lazy vuln generation is skipped for rebuilt nodes, or explicitly document/intend that a rebuilt node is a new system that can have new (fewer/milder) vulns. Ensure detection tier and early-win logic treat a rebuilt node sensibly.

### 6. Zero Day modifier is a complete no-op
**Where:** Defined at index.html:1243, weighted in difficulty scoring at :1293, included in Hard/Nightmare presets at :1259/:1261. No gameplay code references `zeroday_mod`.
**Problem:** The complication "One critical vuln has no available fix" is never implemented. Players enabling it get a harder difficulty score (more intel) for free. (The `zeroday` vuln *category* exists independently and is mitigate-only, but the modifier is supposed to guarantee one critical no-fix vuln.)
**Fix prompt:** Implement it — e.g. after vuln generation for pre-discovered nodes in `startRun`, pick one node on the likely attack path and inject (or upgrade) a `zeroday`-category vuln with severity 9–10 and `fixLocked` semantics stronger than mitigation (or allow mitigate only, per the category). Alternatively remove the modifier from the UI/presets until implemented.

### 7. Insider Threat doesn't establish persistence (contradicts its description)
**Where:** `generateNetwork()` at index.html:2030–2036, `startRun()` at index.html:3027–3035; modifier desc at :1240 ("One node starts compromised with persistence"); GAME_THEORY.md says the same.
**Problem:** The insider node gets `compromised = true`, a foothold, and `compromiseConfirmed = true`, but never `persistence = true` / `persistedNodes.add(...)`. A single IR action fully evicts the "insider" — much weaker than documented. Also `compromisedAt = 0` is falsy, so the node is excluded from the Response score filter (`filter(n => n.compromisedAt)` at :4883), and since the node may be undiscovered (zone 2 is only 60% pre-known), the player can have an invisible confirmed compromise.
**Fix prompt:** Set `persistence = true` and add to `persistedNodes` for the insider node; use `compromisedAt = 0.001` or change the scoring filter to `n.compromisedAt !== undefined`; consider forcing the insider node to be discovered (or discoverable via an early anomaly) so the mechanic is playable.

---

## P1 — Bugs with clear player-visible impact

### 8. Budget can go negative via the rework multiplier
**Where:** `startFix()` at index.html:2863–2873.
**Problem:** Cost is rolled as `rand(lo, min(hi, R.budget))` (capped at current budget), but the 1.5× rework/upgrade multiplier is applied *after* the cap — so re-fixing a mitigated/configured vuln with a full fix can charge up to 1.5× your remaining budget, driving `R.budget` negative.
**Fix prompt:** Apply the 1.5× multiplier before checking/capping against the budget, and reject the action (with the existing "Insufficient budget" log) if the final cost exceeds `R.budget`. Also note the side-effect of the current cap: being poor makes fixes cheaper in expectation — consider whether cost should be independent of budget (roll full range, require affordability) for cleaner economy balance.

### 9. Anomaly and persistence indicator bars never pulse (malformed HTML)
**Where:** `_renderAssetListNow()` at index.html:5253 and :5257.
**Problem:** Both risk-bar layers have **two `class` attributes** on one element: `<div class="asset-bar-fill" id="…" class="anim-pulse-1_5" …>`. HTML parsers ignore the duplicate attribute, so the pulse animation classes are dropped — the amber anomaly bar and red persistence bar render static instead of pulsing.
**Fix prompt:** Merge into a single class attribute: `class="asset-bar-fill anim-pulse-1_5"` (and `anim-pulse-2` for the persistence layer).

### 10. Tutorial/contextual hints break on the second run of a session
**Where:** `checkGameLoopHints()` and module-level `_lastHintCheck` at index.html:1663–1669.
**Problem:** `_lastHintCheck` is never reset when a new run starts. After a run ends at elapsed ≈ 900, a new run starts at elapsed 0, so `now - _lastHintCheck < 1` stays true until the new run's clock passes the old value — suppressing in-loop hints (head-start warning, unscanned-criticals warning) for most of the run.
**Fix prompt:** Reset `_lastHintCheck = 0` inside `startRun()` (alongside the other cache resets at :2925–2926).

### 11. Re-compromise after a reclaim corrupts the Response score
**Where:** `endRun()` response calculation at index.html:4883–4891.
**Problem:** `reclaimedAt` is set on first IR success and never cleared. If the node is compromised again later and never re-reclaimed, `responded - compromisedAt` uses the *old* `reclaimedAt`, which can be earlier than the new `compromisedAt` — producing a negative response time that inflates the score.
**Fix prompt:** Clear `node.reclaimedAt` when the node is compromised (in `ptCompromiseNode`), or track per-incident (compromisedAt, reclaimedAt) pairs and average over incidents. Clamp each incident's response time to ≥ 0.

### 12. Global keyboard shortcuts act on dead runs and wrong screens
**Where:** init keydown handler at index.html:5991–6021; `endRun`/`restartClick` never null `R`.
**Problem:** The handler only checks `if (!R) return`. After a run ends (results screen, or back at HQ via restart), `R` still exists, so Space toggles pause of the finished run (clock element gets blink class), `?` opens the *run* help modal over the HQ screen, `s` attempts scans, 1/2/3 set speeds, and the hire button/Konami cheat still mutate the dead run.
**Fix prompt:** Gate run-scoped shortcuts on the run screen being active (e.g. `document.getElementById('run-screen').classList.contains('active')` or track current screen), and/or set `R = null` when leaving the run screen (restart + results). Keep Escape-closes-modal global.

### 13. Doubling-up double-applies strategic buffs and duplicate logs
**Where:** `maybeDoubleUp()` at index.html:4709–4736; persistence completion at index.html:4644–4672.
**Problem:** The helper pentester copies the primary's state/target/progress but keeps its own timer. When the primary completes persistence, the helper (still in 'persisting' on the same node) also completes a moment later and re-runs the whole completion block: re-adds footholds (guarded), but re-calls `rtApplyStrategicBuff` (incrementing `exploitBonus` a second time, up to the 0.20 cap) and re-logs "Persistence established."
**Fix prompt:** When a persistence/pivot completes, reset any pentester whose `doubledWith` points at the completing pentester (and vice versa), or make the completion block idempotent by checking `rt.persistedNodes.has(target.id)` before applying effects.

### 14. Incarceration splices the pentester array during iteration
**Where:** removal at index.html:4514–4516; iteration in `processRedTeam` at index.html:4788–4791.
**Problem:** `pentesters.splice()` runs inside `updatePentester` while `processRedTeam` iterates `for (const pt of rt.pentesters)` — the pentester after the removed one is skipped that tick. Low impact but a real iteration hazard.
**Fix prompt:** Mark the pentester `incarcerated = true` and filter the array once after the loop in `processRedTeam`.

### 15. Anomaly investigations can never be false positives
**Where:** anomaly creation only at index.html:3306–3309 (scan of compromised node) and :4775–4777 (delayed Tier-2 compromise alert); resolution at index.html:3386–3399.
**Problem:** GAME_THEORY.md sells the anomaly pipeline as alert triage where "not every alert is real," and the resolution code has a false-positive branch — but anomalies are only ever set on genuinely compromised nodes, so investigation always confirms (the FP branch is reachable only in rare eviction races). The triage tension the docs describe doesn't exist.
**Fix prompt:** Occasionally generate benign anomalies — e.g. when a pentester's exploit *fails* on a Tier-2 node, or a small random chance per minute on scanned nodes — so investigations sometimes dismiss. Tune so ~20–30% of anomalies are false positives, and make sure the FP path doesn't clear a real simultaneous compromise.

### 16. IoT / unmanaged devices are treated as high-risk physical targets (contradicts design)
**Where:** `ptPickPhysicalTarget` low-risk filter at index.html:4349–4350 (`zone < 3`); intrusion resolution `isHighRisk = target.zone >= 3` at index.html:4487–4488.
**Problem:** IoT lives in zone 5, so an onsite pentester attacking an IoT device (documented as no-risk reconnaissance, listed as a low-risk target in GAME_THEORY.md) gets the 0.40 high-risk success rate and rolls for incarceration. The zone number is being used as a proxy for physical security level, but zone 5 (Cloud/Shadow IT) isn't physically secured.
**Fix prompt:** Classify physical risk by node type, not zone: `wifiap`, `iot`, `workstation`, `printer`, `unmanaged` are low-risk; zone 3–4 infrastructure (servers, DCs, databases, key vaults) are high-risk. Apply the same classification in both the target picker and the resolution code.

---

## P2 — Game balance

### 17. Efficiency axis is miscalibrated — caps the max achievable score, makes S-grade unattainable
**Where:** `endRun()` at index.html:4893–4899; S-grade gate at :4917.
**Problem:** `efficiency = riskReduced / (budgetSpent/1000) * 10`. A typical fix reduces ~2–6 risk points (severity × exploitability) and costs $2–15K, so realistic efficiency lands around 10–30 — even a perfect run scores ~88–92 total (30+25+15+15 + ~3–5 from efficiency). S requires 95+, so **S-grade is mathematically out of reach** in normal play. It also punishes exactly the expensive-but-correct plays (Replace, Burn & Rebuild) the game otherwise encourages.
**Fix prompt:** Recalibrate efficiency to a relative measure, e.g. `riskReduced / totalAddressableRisk` blended with a spend-ratio factor, or normalize per-difficulty so that a competent run scores 70–90 on this axis. Verify post-change that a flawless run can actually hit 95+ at high difficulty, and that S remains rare.

### 18. S-grade difficulty gate contradicts all documentation
**Where:** index.html:4917 (`finalScore >= 95 && R.diffScore >= 7`); README:108, GAME_THEORY:299, in-game help :867 all say "difficulty 5+".
**Fix prompt:** Pick one (5.0 is friendlier given item 17) and align code + README + GAME_THEORY + help text.

### 19. Difficulty anchors have drifted: "Normal" computes ≈5.1, but 4.3 is the 1.0× intel anchor
**Where:** `computeDifficultyScore()` at index.html:1272–1304; intel multiplier anchor `4.3` at :1757 and :4932; auto-F hard gate `diffScore >= 4` at :4907/:4915.
**Problem:** With the current weights, the Normal preset scores ~5.1 (docs assume ~4.3). Two knock-on effects: (a) Normal players earn ~1.2× intel, so the progression pacing in the README table is off; (b) the auto-F hard gate (difficulty ≥ 4) is active at Normal — a single zone-4 breach at the default difficulty means grade F **and 0 intel**, which is brutal for the middle preset and reads as intended for "difficulty 4+" (i.e., above normal).
**Fix prompt:** Recompute each preset's difficulty score, then either retune weights so Normal ≈ 4.3, or update the anchor constant and the hard-gate threshold (e.g. gate at diffScore ≥ 5.5 so Normal gets the −30 penalty path instead of auto-F). Also consider awarding partial intel on F runs at the gate (e.g. 25% of score) so a 14-minute run isn't a total zero.

### 20. Mitigate dominates the remediation economy
**Where:** FIX_TYPES at index.html:1186–1190; posture scoring at :4871–4877; brute-force reduction at :4132–4133.
**Problem:** Mitigate: always succeeds, cheapest ($1–4K), fast, reduces exploit chance 50–75%, earns posture credit (severity × mitigationFactor), *and* reduces brute-force chance. Patch/Replace risk failure, cost 2–8× more, and failed fixes charge full price then demand 1.5× rework. Combined with item 1 (Configure useless), optimal play collapses to "mitigate everything, patch nothing." The docs promise "a player who only mitigates will find their mitigations eroded over time" — but no erosion mechanic exists.
**Fix prompt:** Implement the documented erosion: mitigations decay (mitigationFactor decreases over 2–4 minutes) or can be defeated by re-scanning pentesters (small chance per exploit attempt that a mitigation is bypassed permanently, with a SIEM log line). Alternatively/additionally reduce posture credit for mitigation (e.g. 50% of mitigationFactor) so full fixes stay attractive.

### 21. Supply Chain complication punishes scanning and is oddly cheap to fix
**Where:** `processModifierTimers()` at index.html:3853–3902.
**Problem:** Targets are drawn only from `discovered && scanned` nodes — a player who scans widely gets 3–5 critical backdoors; a player who scans nothing is immune. That inverts the incentive the game is built on. Also the injected severity 8.5–10 vuln has `fixType: 'replace'` but `cost: rand(800, 1500)` — about 10× cheaper than any normal Replace ($8–15K), so it's trivially cleaned up.
**Fix prompt:** Draw targets from all discovered (or all) nodes regardless of scan state (unscanned affected nodes could surface as anomalies or only be revealed when scanned — that's a nice information-asymmetry beat). Raise the fix cost to something meaningful ($4–8K) or make it a longer fix, and make sure `R.totalVulns` counting stays consistent for unscanned targets.

### 22. Intel/unlock pacing: most templates unlock almost immediately
**Where:** intel formula at index.html:4932; template requirements at :1196–1205.
**Problem:** A single decent Normal run yields ~60–100 intel — that's Enterprise (25) + Manufacturing (50) instantly, and all 8 templates within ~6–8 runs. If templates are meant to be a career-long progression, the curve is far too fast; if they're meant as variety unlocked quickly, the README's framing ("lifetime career score") oversells it.
**Fix prompt:** Decide the intended pacing. If slower: divide intel earnings by ~3 and/or raise late-tier requirements (e.g. 0/50/125/250/450/700/1000/1500). If fast is fine, no code change — just adjust the docs.

### 23. No way to queue actions while paused
**Where:** all action functions early-return on `R.paused` (`scanAsset` :2836, `startFix` :2853, `deepScan` :3531, `startInvestigation` :3569, `burnAndRebuild` :3611, `hireContractor` :2908).
**Problem:** Pause is advertised as "time to think" (pause flavor text even says "plan your next move"), but you cannot issue any commands while paused — you can only look. Whether pause-planning should be allowed is a design decision, but the current mix (encouraged to pause, forbidden to act, and `hireContractor` silently does nothing) feels unintentional.
**Fix prompt:** Either allow issuing actions while paused (they simply don't progress until unpause — most timers already work this way since progress accrues in the game loop), or surface "unpause to act" feedback on the buttons.

### 24. Persistence "5-second re-entry" doesn't exist; hardening actually deters re-entry
**Where:** help text :999 and README:74–75 claim re-entry "in 5 seconds"; actual behavior: persisted nodes just remain in `rt.footholds` after IR (index.html:3413–3416), while the node itself is hardened for 240s (exploit chance ×0.2 at :4118).
**Problem:** The real persistence effect (node stays usable as an expansion source even after eviction) is decent, but there's no fast re-compromise mechanic, and hardening makes the persisted node *harder* to re-enter than the docs promise. Players who burn $8–20K + an analyst on rebuilds are over-paying relative to the actual threat.
**Fix prompt:** Implement documented behavior: if a node has `persistence` and is not compromised, a pentester with that foothold can re-compromise it via a short (≈5s) "re-entry" action that bypasses exploit rolls and hardening. That makes Burn & Rebuild's cost justified. Update docs if you tune the numbers.

---

## P3 — Polish, dead code, doc drift

### 25. Dead code sweep
**Fix prompt:** Remove or wire up: `rtGetInterval()` (index.html:3978, never called), `findPath()` (:4820, never called), `PAUSE_MESSAGES` + `R._pauseMsg` (:5913, :3910 — pause flavor text never displayed; either render it on pause or delete), `cloud_mig` modifier check (:1858, no such modifier), `RUN_MODIFIERS` (:1208), the unreachable head-start probe branch in the pentester scanning state (:4584–4593 — `processRedTeam` returns during head start, so it never runs; this also means the documented "red team probes during head start" flavor never appears — consider making probes actually fire during head start for tension), the tutorial `'passive'` step (unreachable — `endBriefing` jumps straight to `'scan'`), the duplicate `renderAssetList()` call in `scanAsset` (:2848–2849), and the garbage `strokeStyle` construction (:2600, immediately overwritten).

### 26. Documentation / help-text corrections (single prompt)
**Fix prompt:** Sync docs to code (or vice versa):
- Help :982 says "1-3 pentester agents" — slider is 1–5.
- Help :1003 says locked out "30+ seconds" — code is 45s (:4807); README/GAME_THEORY say 45.
- Help :974 claims some vuln categories are "EDR required" (missing monitoring, physical, zero-days) — in code, a vuln scan reveals *all* vulns (:3284); EDR reveals nothing extra. Either implement EDR-gated vuln categories (interesting mechanic, matches GAME_THEORY "EDR finds everything including physical gaps") or fix the text.
- Help :904 lists domain controllers in zone 3 — they're zone 4.
- `strong_iam` desc in README ("Suppress brute force / credential vulns"): code only suppresses brute force (:4129); it does not filter `default_creds` vulns. Implement the vuln filter in `generateVulns` or reword.
- Grades line :867 and README S-grade requirement (see item 18).
- Slider value display: Team/Red-Team Expertise show raw multipliers ("1.5x" is *worst* for the player, "0.6x" is *best*) — confusing. Display as speed ("Slow/Normal/Fast") or invert to a bonus percentage.

### 27. Achievement fixes (single prompt)
**Where:** ACHIEVEMENTS :1335–1348, `checkAchievements()` :4971–4990.
**Fix prompt:**
- `clean_sweep` ("Discover 100% of assets") and `no_stone` ("Find all hidden assets") are the same condition (:4976, :4984) — differentiate (e.g. no_stone = all *shadow IT* found) or merge.
- `zero_breach` unlocks on any run with no zone-4 breach, including Tutorial F-grades (:4977) — require a passing grade and/or non-tutorial difficulty.
- `speed_run` is unobtainable until the early win works (item 4).
- `crit_cond` (:4986) requires every revealed severity-9+ vuln `fixed` — zero-day category vulns are mitigate-only, so one revealed critical zero-day makes it impossible. Count mitigated zero-days (and `fixLocked` vulns) as addressed.
- `odds` desc says "Pass Threat Level 8+" — update wording to the difficulty-score system.

### 28. Minor gameplay nits (single prompt)
- `sfxScan`/`sfxFix` play even when the action is rejected (busy/paused/broke) — hooks at :5728–5731 fire before the guards. Move sounds into the action functions after validation.
- `hireContractor` silently no-ops while paused (:2908) — add feedback or allow it.
- Pinch-zoom is unsupported on touch devices (only one-finger pan, :2313–2326); the wheel handler never fires on mobile, so mobile players are stuck at zoom 1 except via double-tap? Add two-finger pinch handling.
- `processBudgetDrip`'s guard `if (!R || !R.budgetDripTimer) return` (:3738) would permanently stop the drip if the timer ever landed on exactly 0 — use `== null` check instead.
- Passive recon modifier (+20% pre-discovered, :2010–2013) has no effect on zones 0, 1, 4 (already 100% pre-known) — matches the letter of the description but consider noting it only helps Corporate/Internal/Cloud zones.
