# PumaSecure — Vulnerability Reference

Complete reference for all vulnerability categories, their gameplay properties, and how they interact with scanning, fixing, detection, and red team exploitation.

---

## Vulnerability Categories

| # | ID | Name | Fix Options | Attack Vectors | Passive Detect | Detection Impact | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `unpatched` | Unpatched Software | Patch, Replace | Network, Network, Adjacent | No | — | Most common. Double-weighted toward Network vector. |
| 2 | `default_creds` | Default Credentials | Configure | Network, Network, Adjacent | Yes | — | Cheap to fix. Double-weighted toward Network. |
| 3 | `misconfig` | Misconfiguration | Configure | Network, Adjacent | Yes | — | Fast, cheap configure fix. |
| 4 | `crypto_weak` | Encryption Weakness | Configure, Patch | Network, Adjacent | Yes | — | Can configure (fast) or patch (permanent). |
| 5 | `injection` | Injection Flaw | Patch | Network | No | — | Network-only. Patch is the only option. |
| 6 | `auth_bypass` | Authentication Bypass | Patch | Network | No | — | Network-only. Patch is the only option. |
| 7 | `privesc` | Privilege Escalation | Patch, Configure | Local | No | — | Requires same-zone foothold or domain_admin buff. |
| 8 | `exposed_svc` | Exposed Service | Configure | Network, Network, Adjacent | Yes | — | Double-weighted Network. Quick configure fix. |
| 9 | `no_monitor` | Missing Monitoring | Configure | Local, Adjacent | No | **Forces Tier 3** | **Gameplay-critical.** Unfixed = node is blind to attacks regardless of scan state. |
| 10 | `data_expose` | Data Exposure | Configure, Patch | Network, Local | No | — | Mixed vectors. Local requires foothold. |
| 11 | `supply_chain` | Supply Chain Risk | Replace, Mitigate | Network, Adjacent | No | — | No patch available. Replace or mitigate only. |
| 12 | `physical` | Physical Security | Mitigate | Physical | No | — | **Mitigate only.** Affects physical intrusion success/capture rates. |
| 13 | `cloud_misconfig` | Cloud Misconfiguration | Configure | Network | Yes | — | Network-only. Quick configure fix. |
| 14 | `eol` | End-of-Life | Replace | Network, Adjacent, Local | Yes | — | **Replace only.** All three remote vector types. |
| 15 | `zeroday` | Zero Day | Mitigate | Network, Network, Adjacent | No | — | **Mitigate only.** Double-weighted Network. Cannot be fully fixed. |

### Key

- **Fix Options**: Which remediation types are natural for this category. **Mitigate is additionally always available for every category** — it's the universal emergency stopgap, but its effectiveness erodes over ~5 minutes and it must be refreshed or upgraded to a permanent fix.
- **Attack Vectors**: The pool from which a specific vuln's vector is randomly selected (duplicates = higher probability)
- **Passive Detect**: Whether passive scanning can reveal this vulnerability type (6 of 15 categories)
- **Detection Impact**: Special effect on the three-tier detection system

---

## Attack Vector System

Each vulnerability has one attack vector, randomly selected from its category's vector pool. The vector determines both severity weighting and red team access requirements.

| Vector | Severity Weight | Red Team Requirement | Base Exploit Multiplier |
|--------|----------------|---------------------|------------------------|
| **Network** | 1.00 | None (remotely exploitable) | 1.0x exploitability |
| **Adjacent** | 0.75 | None (nearby network segment) | 0.75x exploitability |
| **Local** | 0.50 | Same-zone foothold OR `domain_admin` buff | 0.1x (no foothold) / 0.5x (same zone) / 0.7x (domain_admin) |
| **Physical** | 0.25 | **Pentester must be physically onsite** | 0.15x (onsite base) / 0.25x (onsite + physical_proximity buff) |

### Physical Access Escalation

Physical vector vulnerabilities (unlocked server racks, missing badge access, no cameras) **cannot be exploited remotely** — no amount of network access lets you pick a physical lock. A pentester must be physically onsite to attempt these.

| Condition | Multiplier | How Achieved |
|-----------|-----------|-------------|
| Pentester remote (any foothold) | **Cannot attempt** | Physical vulns are skipped entirely |
| Pentester onsite | 0.15x | Red team dispatches pentester after being stuck 45s+ |
| Pentester onsite + `physical_proximity` buff | 0.25x | Onsite + compromised Wi-Fi AP or IoT in same zone |
| Pentester onsite, high-risk target (zone 3+) | Direct physical penetration | Uses separate physical intrusion mechanic with incarceration risk |

---

## Remediation Options

Each vulnerability category specifies which fix types are available. Some categories have only one option.

| Fix Type | Cost | Time | Success Rate | Downtime | Effect | Available For |
|----------|------|------|-------------|----------|--------|--------------|
| **Mitigate** | $1-4K | 5-15s | 100% | No | Reduces exploitability 50-75% | physical, zeroday, supply_chain |
| **Configure** | $0.5-3K | 3-10s | 85-100% | No | Reduces severity 3-5 points | default_creds, misconfig, crypto_weak, exposed_svc, no_monitor, data_expose, cloud_misconfig |
| **Patch** | $2-15K | 8-30s | 70-95% | Yes | Full removal | unpatched, crypto_weak, injection, auth_bypass, privesc, data_expose |
| **Replace** | $8-15K | 15-30s | 95-100% | Yes | Full removal | unpatched, supply_chain, eol |

### Categories with Only One Fix Option

| Category | Only Fix | Implication |
|----------|---------|-------------|
| Default Credentials | Configure | Cheap and fast, but may be reverted |
| Injection Flaw | Patch | Must accept 70-95% success rate |
| Authentication Bypass | Patch | Must accept 70-95% success rate |
| Missing Monitoring | Configure | Fast fix, but critical to prioritize |
| Physical Security | Mitigate | Can never fully eliminate physical risk |
| Cloud Misconfiguration | Configure | Quick fix available |
| End-of-Life | Replace | Expensive and slow — no shortcuts |
| Zero Day | Mitigate | Can never fully fix — only reduce exploitability |

### Fix Failure and Rework

- Failed fixes set `lastFixFailed = true` and show a "Rework" button
- Reworking uses the same fix type at **1.5x cost**
- A configured vulnerability (severity reduced) can later be patched or replaced for a full fix (also 1.5x cost)
- A mitigated vulnerability can later be addressed with a stronger fix if one is available

---

## Severity Calculation

Vulnerability severity is calculated using a CVSS-inspired formula:

```
baseSev = clamp(AV_WEIGHT * impact * exploitMaturity * 3.6, 0.1, 10.0)
```

Where:
- **AV_WEIGHT**: Network=1.0, Adjacent=0.75, Local=0.5, Physical=0.25
- **impact**: `1 + confidentiality + integrity + availability` (each 0, 0.3, or 0.6; range 1.0-2.8)
- **exploitMaturity**: randomly 0.25, 0.50, 0.75, or 1.0
- **3.6**: scaling constant

### Modifiers

| Modifier | Effect | Condition |
|----------|--------|-----------|
| Legacy system | +1.5 severity | Network modifier "legacy" active (30% chance per node) |
| Threat level | +(TL-1) * 0.15 | Higher threat levels push severity up |

### Severity Bands

| Band | Range | Visual |
|------|-------|--------|
| Critical | 9.0+ | Red badge |
| High | 7.0-8.9 | Orange badge |
| Medium | 4.0-6.9 | Amber badge |
| Low | 0.1-3.9 | Green badge |

---

## Exploitability

Determines the red team's base chance to exploit a vulnerability:

```
exploitability = clamp(exploitMaturity * AV_WEIGHT * (0.7 + threatLevel * 0.03), 0.05, 0.95)
```

### Red Team Exploit Modifiers

The base exploitability is modified during exploitation by:

| Modifier | Effect | Source |
|----------|--------|--------|
| Mitigation | chance *= (1 - mitigationFactor) | Mitigated vulnerability (50-75% reduction) |
| Exploit bonus | chance += exploitBonus | Cumulative from strategic compromises (0-0.20) |
| Bypass perimeter | chance *= 1.3 | Firewall compromised, target is zone 0-1 |
| Tunnel access | chance *= 1.1 | VPN compromised, target is zone 3+ |
| Priv Escalation phase | chance *= 1.15 | Game is in Privilege Escalation phase |
| Exfiltration phase | chance *= 1.25 | Game is in Exfiltration phase |
| Hardened node | chance *= 0.2 | Node was reclaimed via IR (lasts 4 minutes) |
| Deep scanned | chance *= 0.5 | Node was deep scanned by defender |

### Brute Force Fallback

If no individual vulnerability is exploitable, the red team can attempt brute force:

```
bruteChance = 0.05 * threatLevel / 5
```

Modified by: hardened (0.1x), deep scanned (0.5x), mitigated ratio (up to -60%).

---

## Vulnerability Generation

### Count Per Node

```
count = clamp(floor(criticality * 0.8 + random(1, 3)), 1, 7)
```

Where criticality is zone-based: Perimeter=1, DMZ=2, Corporate=2, Internal=3, Critical=5, Cloud=3.

| Zone | Base Criticality | Typical Vuln Count |
|------|-----------------|-------------------|
| Perimeter (0) | 1 | 1-3 |
| DMZ (1) | 2 | 2-4 |
| Corporate (2) | 2 | 2-4 |
| Internal Services (3) | 3 | 3-5 |
| Data / Critical (4) | 5 | 5-7 |
| Cloud / Shadow IT (5) | 3 | 3-5 |

### Category Selection

Categories are selected uniformly at random from the 15 available types. There is no zone-specific filtering — any category can appear on any node type.

---

## Passive Detection

Six of 15 vulnerability categories can be detected by passive scanning without assigning an analyst:

| Passively Detectable | Not Passively Detectable |
|---------------------|-------------------------|
| Exposed Service | Unpatched Software |
| Encryption Weakness | Injection Flaw |
| Default Credentials | Authentication Bypass |
| Misconfiguration | Privilege Escalation |
| Cloud Misconfiguration | Missing Monitoring |
| End-of-Life | Data Exposure |
| | Supply Chain Risk |
| | Physical Security |
| | Zero Day |

Passive detection uses escalating intervals per node: the first vulnerability is revealed after 15 seconds, the second after 35s, then 60s, 90s, 120s. Only discovered but unscanned nodes receive passive scanning.

---

## Detection Tier Interaction

The `Missing Monitoring` vulnerability (`no_monitor`) has a unique gameplay role:

| Node State | Missing Monitoring Unfixed | Detection Tier |
|-----------|---------------------------|---------------|
| Unscanned | N/A | Tier 3 (silent) |
| Scanned | Yes | **Tier 3 (silent)** |
| Scanned | No (fixed or mitigated) | Tier 2 (delayed alerts) |
| Deep scanned | Yes | **Tier 3 (silent)** |
| Deep scanned | No (fixed or mitigated) | Tier 1 (instant alerts) |

This means a deep-scanned node with an unfixed Missing Monitoring vulnerability is **still blind** to red team activity. Fixing this vulnerability should be a top priority on any scanned node.

---

## Example Vulnerability Names

Each category draws from a thematic name pool:

| Category | Example Names |
|----------|--------------|
| Unpatched Software | Unpatched OpenSSL, Log4j 2.x present, Missing Windows KB, Outdated Apache httpd |
| Default Credentials | Default admin password, Factory SSH keys, Default SNMP community |
| Misconfiguration | Open debug port, Permissive firewall rule, Directory listing enabled |
| Encryption Weakness | TLS 1.0 enabled, Expired certificate, Weak cipher suites |
| Injection Flaw | SQL injection endpoint, Command injection vector, Stored XSS |
| Authentication Bypass | Broken session management, JWT none algorithm, Missing MFA |
| Privilege Escalation | SUID binary exploit, Docker socket exposed, Sudo misconfig |
| Exposed Service | Telnet open, RDP exposed, Redis unauth, MongoDB no auth |
| Missing Monitoring | No logging configured, SIEM agent missing, Audit logs disabled |
| Data Exposure | Plaintext passwords in config, Public S3 bucket, Sensitive data in logs |
| Supply Chain Risk | Vulnerable npm package, Compromised pip dependency, Outdated container base image |
| Physical Security | Unlocked server rack, No badge access, Missing security cameras |
| Cloud Misconfiguration | Overpermissive IAM role, Public cloud storage, Missing encryption at rest |
| End-of-Life | Windows Server 2008, Ubuntu 14.04 LTS, Python 2.7 runtime, PHP 5.6 |
| Zero Day | Undisclosed RCE, Zero-day auth bypass, Novel deserialization exploit |

---

## Strategic Priority Guide

Based on the mechanics above, here's how vulnerability categories stack up for gameplay:

### Fix First (Gameplay-Critical)

| Category | Why |
|----------|-----|
| **Missing Monitoring** | Unfixed = you're blind to attacks on that node. Cheap configure fix. |
| **Zero Day** | Can only mitigate, never fully fix. High Network exploitability. |
| **Physical Security** | Can only mitigate. Affects physical intrusion success rates on critical nodes. |

### Fix Early (High Impact)

| Category | Why |
|----------|-----|
| **Unpatched Software** | Most common, heavily Network-weighted, feeds red team exploit chains. |
| **Default Credentials** | Trivially exploitable. Cheap configure fix eliminates the risk. |
| **Exposed Service** | Double-weighted Network vector. Easy target for red team. |
| **Authentication Bypass** | Network-only, high exploitability. Patch is the only option. |

### Fix When Resources Allow

| Category | Why |
|----------|-----|
| **End-of-Life** | Replace-only (expensive, slow). All three vector types. |
| **Supply Chain Risk** | Replace or mitigate only. No patch available. |
| **Privilege Escalation** | Local-only — requires foothold to exploit. Lower priority if perimeter is held. |
| **Data Exposure** | Mixed vectors. Important but less immediately dangerous. |

### Lower Priority

| Category | Why |
|----------|-----|
| **Misconfiguration** | Quick configure fix, moderate severity. |
| **Encryption Weakness** | Adjacent/Network. Configure or patch. |
| **Cloud Misconfiguration** | Network-only. Quick configure fix. |
| **Injection Flaw** | Network-only but single vector. Patch-only. |
