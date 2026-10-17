# PumaSecure

**Fifteen minutes to secure a network before the red team owns it.** An infosec roguelike with a CRT terminal look. A security team dropped into a poorly managed network scans assets, discovers vulnerabilities and fixes them before an AI red team punches through to the critical infrastructure.

**Play:** [greykit.com/games/pumasecure](https://greykit.com/games/pumasecure/)

![HQ — configure your engagement before each run](screenshots/hq-screen.png)

## How it plays

The network is procedurally generated in six zones, from the perimeter in to the data and critical core, plus cloud and shadow IT. One to five pentester agents scan, exploit, persist and pivot on their own, sharing intel as far as their coordination allows, and go onsite when locked out remotely. You see only what your monitoring sees: EDR gives instant alerts, a vuln scan gives delayed anomalies to investigate, and an unscanned node gives nothing. Every scan, fix, investigation and rebuild takes an analyst. Fixes run from fast mitigations that erode to slow replacements, and any of them can fail. Incident response that finishes before the attacker persists evicts them; after that only a Burn & Rebuild clears the node, and it costs an analyst for good.

![Mid-game — scan assets, discover vulnerabilities, and fix them before the red team gets in](screenshots/gameplay-mid.png)

The run is graded S to F on containment, security posture, assessment coverage, incident response and efficiency, and a breached critical asset at difficulty 6+ is an automatic F. Grades earn Intel, a career score that unlocks eight network templates. Five presets run from Tutorial to Nightmare, or a custom mix of sliders, advantages and complications.

![Post-engagement debrief — performance assessment across five scoring axes](screenshots/results-screen.png)

## Keys

Click a node to select it and act on it from the detail panel; double-click runs its quick action. Scroll or pinch zooms the map, drag pans it, `Space` pauses and `?` opens help.

## Saves

Intel, unlocked templates, achievements, stats and settings are kept in the browser's `localStorage` under `pumasecure_save_v1`.

## Credits

Design notes: [GAME_THEORY.md](GAME_THEORY.md), with the vulnerability categories in [VULNERABILITIES.md](VULNERABILITIES.md).

By Matthew "Shan" Romanek, with Claude. Built by [greykit.com](https://greykit.com/).

Made for my own play. Nothing here is offered: no support, no updates, no licence.
