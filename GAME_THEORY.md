# PumaSecure — Game Theory & Design

## Premise

You've been hired to secure a network that has never been properly maintained. You have limited time, limited headcount, and a tight budget. An adversary is coming. Your job is to prioritize skillfully.

This document describes the interlocking systems that make PumaSecure a game about real security decision-making, not reflex or memorization.

---

## The Core Tension: Information Asymmetry

PumaSecure is built on a fundamental asymmetry: **the defender doesn't know what they don't know**.

The player starts with a partial asset list — some nodes are visible, others are undiscovered. Little is known about any system beyond its hostname and function. Vulnerabilities are hidden until actively sought. Compromises are invisible on systems without monitoring.

The red team starts with even less — they can only see the network perimeter — but they have time, patience, and the advantage that the defender must protect everything while the attacker only needs to find one way in.

This creates a game where **the most important resource isn't money or headcount — it's information**, and every action is a bet on what matters most right now.

---

## The Defender's Toolkit

### Discovery Pipeline

Information flows through three tiers, each costing more time and analyst capacity:

**Passive Monitoring** reveals a trickle of easily-detectable issues — exposed services, default credentials, weak crypto, end-of-life software. It's free but slow and incomplete. A player relying solely on passive scanning will miss the vulnerabilities that matter most.

**Vuln Scanning** (active scanning) runs a vulnerability scanner like Nessus or Qualys. It reveals every vulnerability on the host and discovers neighboring assets connected to it. It provides the information needed to prioritize, but costs an analyst slot for 15-25 seconds — time that analyst isn't fixing anything.

**EDR Deployment** (deep scanning) installs an endpoint detection agent like CrowdStrike. It finds everything, including physical security gaps. It also reduces the host's attack surface (reflecting the value of an up-to-date asset inventory and threat model) and provides Tier 1 instant-alert detection. But it's the most expensive scan in terms of time.

The design tension: **scanning gives you information, but costs the same resource (analyst time) that you need to act on that information**. Scanning everything before fixing anything means the red team gets a head start. Fixing without scanning means you might miss the critical vulnerability.

### Remediation Spectrum

Four fix types sit on a spectrum from fast-and-fragile to slow-and-permanent:

| Action | Time | Cost | Success Rate | Effect |
|--------|------|------|-------------|--------|
| **Mitigate** | 5-15s | $1-4K | 100% | Reduces exploitability 50-75%; erodes to ~35% strength over ~5 min |
| **Configure** | 3-10s | $0.5-3K | 85-100% | Reduces severity, may be reverted by operations |
| **Patch** | 8-30s | $2-15K | 70-95% | Full fix, but change windows get denied, dependencies break |
| **Replace** | 15-30s | $8-15K | 95-100% | Near-guaranteed fix, but migrations crash, licenses reject |

These aren't abstract game mechanics — they mirror real security operations:
- **Mitigate** is the WAF rule or firewall ACL you throw up in an emergency. It buys time but doesn't solve the problem.
- **Configure** is the hardening guide applied in production. Fast, but IT ops may revert it during the next maintenance window.
- **Patch** is the vulnerability patch that requires a change window, breaks a dependency, or gets pushback from the application team.
- **Replace** is the full system rebuild or major version upgrade. Expensive, disruptive, but definitive.

Fixes can fail for realistic reasons — operational pushback, unskilled engineers, human error. A failed fix can be reworked (at 1.5x cost), or the player can try a different remediation strategy. A mitigated vulnerability can later be properly patched when resources free up.

The design tension: **the fastest fixes are the least effective, and the most effective fixes take the longest and cost the most**. A player who only mitigates will find their mitigations eroded over time. A player who only replaces will run out of time and money.

### Analyst Capacity

Every action — scanning, fixing, investigating, rebuilding — requires an analyst. Everything competes for the same limited pool of human attention.

This creates constant triage decisions:
- Do you scan the next node or fix the critical vuln you already found?
- Do you investigate the anomaly or keep patching?
- Do you hire the $10K contractor now or save the budget for replacements later?

---

## The Red Team as Independent Agents

The red team isn't a difficulty slider or a random event generator. It's modeled as independent actors with their own resources, information, and decision-making.

### Red Team Resources

The red team has **pentesters** — independent operators that work in parallel. Their count is controlled by the Red Team Size slider (1-5). Each pentester is an autonomous agent with their own target, progress timer, and state. Multiple pentesters can attack different nodes simultaneously, creating multi-front pressure that a single defender must triage.

Pentesters can also **double up** on high-value targets (zone 3+), halving the time to establish persistence on critical infrastructure.

### Red Team Coordination

The **Red Team Coordination** slider controls how pentesters share intelligence:

| Setting | Behavior |
|---------|---------|
| **None** | Each pentester only uses footholds they personally established |
| **Low** | Personal footholds always available; team footholds shared 25% of the time |
| **Mid** | 50% chance of sharing team footholds |
| **High** | 75% chance of sharing |
| **Full** | All pentesters share all footholds — a single foothold is an entry for everyone |

This creates a meaningful difficulty axis: at low coordination, eliminating one pentester's foothold doesn't help the others, but they also can't leverage each other's work. At full coordination, a single breach becomes a team-wide attack vector.

### Multi-Step Attack Chains

The red team doesn't instantly compromise nodes. Each attack follows a realistic chain:

```
IDLE -> SCANNING (5-12s) -> EXPLOITING (4-10s) -> COMPROMISED -> PERSISTING (15-30s) or PIVOTING (3-8s)
```

**Scanning** probes the target for exploitable vulnerabilities. The pentester learns vulnerability categories on the target, building intelligence for the exploit attempt.

**Exploiting** attempts to leverage a known vulnerability. Success depends on the vulnerability's exploitability, the node's defensive posture (hardening, EDR deployment, patching), and accumulated strategic buffs.

**Persisting** establishes a durable foothold on high-value targets. This takes significant time (15-30s), during which the defender can race to interrupt via incident response. Once persistence is established, the foothold survives eviction — the red team can re-enter without re-exploiting.

**Pivoting** is a quick lateral move through low-value targets (IoT, printers, workstations). It's fast but fragile — if the source node is reclaimed, the pivot chain breaks.

### Strategic Buffs

Compromising specific node types grants lasting advantages:

| Node | Buff | Effect |
|------|------|--------|
| Domain Controller | `domain_admin` | +15% exploit chance, local attack vectors without proximity |
| VPN Gateway | `tunnel_access` | Can bypass network topology to reach deep zones |
| Firewall | `bypass_perimeter` | 1.3x exploit chance on perimeter nodes |
| DNS Server | `dns_hijack` | +5% exploit bonus |
| CI/CD Server | `supply_chain` | +8% exploit bonus |
| Load Balancer | `reveal_backends` | Reveals 1-2 hidden backend nodes |
| Wi-Fi AP / IoT | `physical_proximity` | Enables physical attack vectors in same zone |
| Mail / File / Key Vault | `credential_harvest` | +3% exploit bonus per system |

This means the red team's target selection is strategic, not random. A VPN gateway isn't just a perimeter node — it's a skeleton key to the internal network. A domain controller isn't just a high-value target — it's a force multiplier for every subsequent attack.

### Event-Driven Behavior

The red team has no global phase system tied to a clock. Instead, each pentester independently adapts to their situation:

- **No footholds**: Probe and attack perimeter nodes, looking for an entry point
- **Has foothold**: Expand inward, prioritizing high-value and deeper-zone neighbors
- **Deep footholds**: Aggressively target critical assets (zone 4), increased exploit bonuses
- **Locked out**: Regroup and re-target the perimeter — the red team never gives up

The only clock-tied element is a configurable **head start** (1-5 minutes) before pentesters activate, giving the player time to scan and begin remediation.

If the red team is completely locked out (no viable targets for 45+ seconds), they escalate to physical intrusion.

---

## The Detection Game

PumaSecure's detection system models the reality that **you can only defend what you can see**.

### Three-Tier Monitoring

Detection quality depends entirely on the node's security posture:

| Tier | Condition | Scanning Alerts | Compromise Visibility |
|------|-----------|----------------|----------------------|
| **Tier 1** | EDR deployed | Immediate, specific ("Scanning detected on FW-01") | Instant red flag, IR available immediately |
| **Tier 2** | Vuln scanned (no EDR) | Delayed (5-10s), vague ("Unusual traffic in DMZ zone") | Amber anomaly indicator — must investigate to confirm |
| **Tier 3** | Unscanned or missing monitoring | None | Silent — player is completely blind |

A node with an unfixed "Missing Monitoring" vulnerability behaves as Tier 3 regardless of scan state. This makes monitoring gaps a gameplay-critical vulnerability — fixing the RCE is important, but if you can't see the attack, the RCE fix might come too late.

### The Anomaly Pipeline

When a Tier 2 node is compromised, the player doesn't see a breach — they see an **anomaly**. The node glows amber. Something is wrong, but what?

The player must assign an analyst to **investigate the anomaly** (5-10s). The investigation either confirms a compromise (enabling full incident response) or dismisses it as a false positive. This models the real-world friction of alert triage — every anomaly costs attention, and not every alert is real.

### Scanning Reveals Hidden Compromises

If a node was silently compromised (Tier 3) and the player later scans it, the scan discovers indicators of compromise. EDR deployment auto-confirms the breach. This creates a discovery mechanic where **the act of scanning can reveal that you've already been owned**.

---

## Incident Response: The Race

When a compromise is confirmed, the player can assign an analyst to incident response (10-20s). But IR isn't a guaranteed win — it's a **race against the pentester's current activity**.

If the pentester is establishing persistence:
- **IR wins first**: Attacker evicted, node hardened for 4 minutes, pentester reassigned
- **Persistence wins first**: Attacker has a durable foothold. IR still evicts the pentester, but the persistence marker remains — the red team can re-enter in 5 seconds without re-exploiting

If the pentester is pivoting:
- **IR wins first**: Pivot fails, downstream targets safe, pentester reassigned
- **Pivot completes first**: Lateral movement succeeded, but the source is now reclaimed — if the source had no persistence, the downstream access breaks

### Burn & Rebuild

When persistence is established and survives IR, the player has one option: **Burn & Rebuild**. This is the BCP/DR nuclear option — scorched-earth replacement of the compromised system.

- **Cost**: $8K-$20K depending on the node's zone and criticality
- **Duration**: 25-45 seconds (requires an analyst)
- **Effect**: The node goes offline, then comes back as a clean system with zero vulnerabilities. All red team persistence is destroyed. Downstream nodes that depended on this path lose their access (unless they have their own persistence).
- **Penalty**: Costs one analyst permanently — someone's getting fired for letting it get this bad.

The design tension: **Burn & Rebuild is expensive, slow, and permanently reduces your team, but it's the only way to truly eliminate persistence**. The red team knows the clock is ticking — they must race to establish their own persistence deeper in the network before the choke point burns.

---

## Physical Intrusion: The Nuclear Option

When the red team is locked out of the network (perimeter fully patched, no footholds for 45+ seconds), they send a pentester onsite.

### Phase 1: Coffee Shop

The pentester arrives at the physical location (10-15s travel time). From the coffee shop, they can:
- Do everything they could do remotely (scan, exploit perimeter)
- **Plus** directly attack Wi-Fi APs and wireless workstations
- Wi-Fi attacks carry **no special penalty** if they fail — it's low-risk reconnaissance

This models the reality that physical proximity opens entirely new attack surfaces. A locked-down firewall means nothing if the Wi-Fi AP in the lobby is running factory firmware.

### Phase 2: Physical Penetration

From onsite, the pentester may attempt direct physical access to a system:

**Low-risk targets** (Wi-Fi APs, unattended workstations, IoT devices): No incarceration risk. Normal physical attack vector. Failed attempt means trying again later.

**High-risk targets** (server rooms, data centers — zone 3+): Cameras, mantraps, security guards. Success (40-70%) means total control — USB-jack, direct console access, instant persistence. Failure means a roll for detection:
- If physical security vulnerabilities are **unfixed**: 60% chance the pentester is caught
- If physical security is **patched**: 75% chance of capture
- **Caught = permanently removed** from the engagement

### Physical Access Requirements

Physical-vector vulnerabilities (unlocked server racks, missing badge access, no security cameras) are **only exploitable by onsite pentesters**. Having a same-zone foothold through network exploitation does not grant physical access — you need a body in the building. This creates a genuine risk/reward calculation for the red team and a genuine incentive for the player to fix physical security vulnerabilities on critical infrastructure.

---

## The Difficulty System

PumaSecure's difficulty is fully configurable through 8 sliders and 16 modifiers.

### Sliders

**Player Resources** (higher = easier):
- Starting Budget ($15K-$60K)
- Budget Drip ($3K-$12K per 60s)
- Starting Analysts (1-5)
- Team Expertise (0.6x-1.5x action speed)
- Head Start (1-5 minutes before red team activates)

**Red Team Power** (higher = harder):
- Red Team Size (1-5 pentesters)
- Red Team Expertise (0.6x-1.4x attack speed)
- Red Team Coordination (None-Full foothold sharing)

### Modifiers (Advantages)

Each has three positions: OFF / ? (random) / ON.

| Modifier | Effect |
|----------|--------|
| MSSP Monitoring | No "missing monitoring" vulnerabilities |
| Remote Attacker | No physical vulns, no onsite attacks |
| Grizzled IT Veteran | Fixes never fail |
| Strong IAM | Suppress brute force / credential vulns |
| Threat Intel Feed | Reveals the current target of one pentester |
| Network Mapper | Zone outlines visible on the map at start |
| Passive Recon | Start with 20% more assets pre-discovered |
| Incident Playbooks | First remediation on each asset is free |

### Modifiers (Complications)

| Modifier | Effect |
|----------|--------|
| Government Oversight | Fix times 1.5x longer (paperwork) |
| Governance Requirements | 30% of vulns can only be mitigated |
| Insider Threat | One node starts compromised with persistence |
| Pandemic Protocol | One analyst goes AFK periodically |
| Legacy Env | 30% of nodes get legacy flag, boosting vuln severity |
| Zero Day | One critical vuln has no available fix |
| Shadow IT | 2-3 unmanaged devices appear mid-game — red team discovers them first |
| Supply Chain | Backdoored dependency spawns same critical vuln on 3-5 nodes simultaneously |

Random modifiers are weighted by impact — powerful modifiers (Grizzled Veteran, Insider Threat) have lower activation chances than mild ones (Pandemic, Governance). Random rolls slightly favor the player.

### Difficulty Score and Intel

Every configuration produces a difficulty score (1.0-10.0). This score determines the **Intel multiplier** for that run. The multiplier is anchored to the Normal preset — Normal always yields exactly 1.0x intel, and the anchor is computed from the preset itself so retuning presets keeps it honest. Higher difficulty yields more; lower yields less. Failed runs earn a 25% consolation rather than zero.

Five presets (Tutorial through Nightmare) configure everything, or players can use Custom to dial in exact parameters.

---

## Scoring: What Gets Measured

The final score reflects five dimensions of security program effectiveness:

| Axis | Weight | Measures |
|------|--------|----------|
| **Containment** | 30% | What percentage of nodes were never compromised (or successfully reclaimed)? |
| **Security Posture** | 25% | What percentage of known vulnerability severity was addressed? |
| **Assessment Coverage** | 15% | What percentage of total vulnerabilities were discovered? |
| **Incident Response** | 15% | How quickly were compromises detected and remediated? |
| **Efficiency** | 15% | How much risk was reduced per dollar spent? |

**Hard gate (difficulty 6+)**: Any critical asset (zone 4) breached = automatic **F**, score capped at 39. At lower difficulties (including the Normal preset), a critical breach incurs a -30 penalty but isn't an automatic failure.

**Early win**: If the player secures all discovered nodes, eliminates all compromises, and the red team has no footholds after 5 minutes, the engagement ends early. The response score uses actual elapsed time, rewarding decisive play.

### Grades

| Grade | Score | Requirement |
|-------|-------|-------------|
| **S** | 95+ | Difficulty 6+ |
| **A** | 85+ | Strong defense |
| **B** | 70+ | Functional defense |
| **C** | 55+ | Needs improvement |
| **D** | 40+ | Poor posture |
| **F** | <40 | Failed, or critical breach |

---

## Career Progression

Intel earned per run is a **lifetime career score** that unlocks new network templates:

| Template | Intel Required |
|----------|---------------|
| Small Office | 0 (free) |
| Enterprise | 40 |
| Manufacturing | 100 |
| University | 180 |
| Hospital | 280 |
| Startup | 420 |
| Fintech | 600 |
| Government | 850 |

Higher difficulty yields more intel, incentivizing players to increase the challenge as they improve. Each template provides a different network topology and asset mix, requiring different strategies.

---

## Design Philosophy

PumaSecure is not a tower defense game with a security skin. Every mechanic maps to a real security concept:

- **Vuln Scanning** = vulnerability assessment and asset discovery
- **EDR Deployment** = endpoint detection and response rollout
- **Fixing** = patch management, configuration hardening, system replacement
- **Detection tiers** = the SIEM coverage gap
- **Anomaly investigation** = alert triage and false positive management
- **Persistence vs pivot** = APT tradecraft (C2 implants vs living-off-the-land)
- **Burn & Rebuild** = business continuity and disaster recovery
- **Physical intrusion** = physical penetration testing and facility security
- **Analyst slots** = the staffing shortage that every security team faces
- **Budget** = the reality that security competes with every other business priority
- **Shadow IT** = unmanaged devices appearing on corporate networks
- **Supply Chain** = dependency compromise (SolarWinds, Log4Shell, XZ Utils)
- **Red Team Coordination** = the difference between a solo operator and a coordinated APT group

The game teaches by forcing trade-offs that mirror real security leadership: you will never have enough time, people, or money to fix everything. The question is always **what do you fix first, and what do you accept the risk on?**
