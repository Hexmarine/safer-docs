# Upstream Technical Documents Review

- Date: 2026-04-12
- Scope: cross-reference 11 new upstream technical documents against prior investigation findings; identify confirmations, contradictions, and new facts
- Related systems: MQTT, firmware/devices, DNS/hosting, feature flags, SIM connectivity, infrastructure hardening, development process

## Objective

Eleven PDFs were added to `docs/upstream/Technical Documents/`. This note reviews each against prior investigations and records what they confirm, contradict, or add.

## Documents Reviewed

| File | Topic |
| --- | --- |
| `SENSORGLOB-Firmware & Protocol Specification` | Full MQTT command set, hardware specs, serial/model codes |
| `SENSORGLOB-Event types` | Business-readable event names, human-readable message templates |
| `SENSORGLOB-Domains & Hosting` | All production, QA, sandbox, dev URLs; MQTT port; data storage locations |
| `SENSORGLOB-Device Types Documentation` | Client device types (Android=1, iOS=2, Web=3); marked WIP |
| `SENSORGLOB-Feature Flags` | 25 LaunchDarkly flags with per-environment status |
| `SENSORGLOB-KORE Super SIM` | Carrier preference, SIM lifecycle, data thresholds |
| `SENSORGLOB-Infrastructure Hardening Standards` | Network rules, SSH IP whitelist, OS baseline, patch SLAs |
| `SENSORGLOB-Git Branching Strategy` | Current GitFlow + two planned future strategies |
| `SENSORGLOB-Asset Naming Standards` | Web/media file naming conventions |
| `SENSORGLOB-Developer Code of Conduct` | Sprint cadence, Jira workflow, communication channels |
| `SENSORGLOB-Beta Test Plan` | 2021/2022 historical beta context |

---

## Confirmations

These docs align cleanly with what prior investigations found.

| Upstream doc | Confirmation | Prior note |
| --- | --- | --- |
| Firmware spec | MQTT topics `sg/sas/cmd/{sn}` and `sg/sas/resp/{sn}` are exactly as observed in code | `2026-04-09-mqtt-device-communications.md` |
| Firmware spec | Hub is the only cloud-facing MQTT client; alarms relay through the hub over 433 MHz FSK mesh | `2026-04-09-platform-architecture-and-data-flow.md` |
| Firmware spec | VERIFY→ALARMS→BATTERY synchronisation chain is explicitly how the hub reports state after connect | `2026-04-09-alarm-event-traces.md` |
| Firmware spec | QR codes encode `https://activation.sensorglobal.com/?s=<serial>` exactly | `2026-04-09-platform-architecture-and-data-flow.md` |
| KORE SIM doc | Hub uses LTE Cat-M backhaul via KORE (formerly Twilio) Super SIM | `2026-04-09-platform-architecture-and-data-flow.md` |
| KORE SIM doc | KORE was previously Twilio's Super SIM product before the IoT asset sale | `2026-04-09-mqtt-device-communications.md` |
| Domains doc | AWS ap-southeast-2 (Sydney) is the primary data region | `2026-04-09-ap-southeast-2-architecture-baseline.md` |
| Domains doc | All production portal URLs (`admin`, `agent`, `trader`, `owner`, `activation`) match Route 53 findings | `2026-04-09-prod-external-account-migration-evaluation.md` |
| Domains doc | `mqtt.sensorglobal.com` and per-environment MQTT hostnames match architecture baseline | `2026-04-09-ap-southeast-2-architecture-baseline.md` |
| Domains doc | Twilio communication services are hosted in the United States (not Australia); affects data residency | `2026-04-09-platform-architecture-and-data-flow.md` |
| Infra Hardening | OpenVPN instances in every environment are intentional per policy | `2026-04-09-ap-southeast-2-architecture-baseline.md` |
| Infra Hardening | AWS Console is the stated management plane; no IaC mentioned — consistent with EC2/CodeDeploy pattern | `2026-04-09-ap-southeast-2-architecture-baseline.md` |

---

## Contradictions and Confirmed Gaps

### 1. QoS mismatch — formally confirmed by firmware spec

**What the spec says:** "All commands will use QoS 2 – Exactly Once unless otherwise specified."

**What we found in code:** the backend MQTT subscriber subscribes at QoS 0 and its generic publish helper does not set an explicit QoS override.

**Status before this review:** flagged as a suspected inconsistency in `2026-04-09-mqtt-device-communications.md`.

**Status after this review:** confirmed. The firmware spec is the authoritative source for the intended delivery guarantee. The backend implementation does not match it. The risk is that hub events may be delivered more than once or not at all under network instability, without the exactly-once guarantees the spec relies on.

---

### 2. LOW_BATTERY — spec defines three hub thresholds; backend handles only the 15% crossing

**What the spec and Event Types doc say:**
- Hub LOW_BATTERY triggers at **15%, 10%, and 5%** (three re-alerts as battery continues to drain).
- Alarm LOW_BATTERY triggers at **5% only**.
- `LOW_BATTERY` is a formally defined standalone event in the firmware command set.

**What we found in code:** the backend MQTT subscriber hits an early `return` on a standalone `LOW_BATTERY` command with no further processing. The only low-battery path that works is `BATTERY <= 15` inside the `BATTERY` branch — which catches the first threshold for hubs but does not generate re-alerts at 10% and 5%.

**Status before this review:** flagged as suspected in `2026-04-09-alarm-event-traces.md`.

**Status after this review:** confirmed implementation gap. A hub battery draining from 15% to 10% to 5% would emit `LOW_BATTERY` events for the 10% and 5% crossings, and those would be silently discarded. Operators and tenants would not receive the second or third low-battery warning even though the firmware spec requires them.

---

### 3. FAULT — spec defines a real event; backend has no handler

**What the spec says:** `FAULT` is emitted by the controller or a linked alarm when a self-diagnosed fault occurs. Current defined fault codes:
- **10: Cleaning required** — emitted when the sensing chamber is obstructed and can no longer adequately detect smoke.

**What we found in code:** `FAULT` appears in the protocol summary and in `firmware.md`, but there is no `FAULT` handler in the main MQTT subscriber dispatch path.

**Status before this review:** noted as possibly dormant or handled elsewhere in `2026-04-09-alarm-event-traces.md`.

**Status after this review:** confirmed as an implementation gap. If a smoke alarm's sensing chamber becomes blocked and it emits `FAULT CODE 10`, the backend silently discards the event. No notification, no maintenance job, no audit trail.

---

### 4. HUSH and ALERT STATE encoding — spec uses 1=On, 2=Off (not 0=Off)

**What the spec says:** for both `HUSH` and `ALERT`, `BoolState` is:
- 0 = Undefined
- 1 = On
- 2 = Off

**Implication for the backend:** clearing an `ALERT` sends `STATE:2`, not `STATE:0`. If any backend branch checks `STATE === 0` or treats `!== 1` as "off", it would behave incorrectly on clear events. This is worth verifying in the MQTT subscriber's alert-off and hush-off paths.

**Status:** open question. Prior event traces showed `STATE:1` for on-events but did not explicitly confirm the off-path handling of `STATE:2`.

---

### 5. Firmware spec still references "Twilio Super SIM" — branding is stale

**What the spec says:** "Twilio Super SIM for connectivity."

**What the KORE SIM doc says:** "We use KORE (previously Twilio) Super SIM."

**Impact:** the two upstream documents are out of sync. The firmware spec should be updated to reference KORE, not Twilio, so new team members are not confused about the SIM provider.

---

### 6. Device Types doc describes web portal client types, not IoT hardware types

**What the Device Types doc says:** three device types exist: Android (1), iOS (2), Web (3). These are the client platforms for the SaaS web portal. The doc is marked WIP. It says "the connection between the system and the backend is secured through a VPN."

**Potential confusion:** the firmware spec defines hardware device model codes (C001=Hub, A001=Smoke Alarm, A004=Water Leak, etc.). The Device Types doc covers something completely different — user portal session device types. These should not be conflated.

**VPN claim:** the claim that web app clients connect via VPN is inconsistent with our finding that all portals are served through public CloudFront distributions and ALBs over HTTPS. The statement likely refers to the backend-to-database or internal connectivity, not the end-user browser connection.

---

## New Information Not Previously Documented

### MQTT broker port: 18756

The Domains doc lists `mqtt.sensorglobal.com` with port **18756** and the same for all environment MQTT hostnames. This port is not a standard MQTT port (standard is 1883 or 8883 for TLS). It was not recorded in any prior investigation and should be noted in network/firewall analysis.

### Device model code table

From the firmware spec serial number format:

| Code | Model |
| --- | --- |
| C001 | 217E-C001 Sensor Hub (AU) |
| A001 | 217E-02 Photoelectric Smoke Alarm (AU) |
| A002 | Carbon Monoxide Alarm |
| A003 | 217E-03 Photoelectric Smoke Alarm + Carbon Monoxide Alarm (US) |
| A004 | 217E-W01 Water Leak Sensor |
| A005 | 217E-W02 Water Shutoff Valve |

These codes are embedded in the 20-character serial number at positions 13–16 (the 4-character model segment).

### Serial number structure

All device serials are 20 characters: `XXXXXXXXXXXX-XXXX-XXXX` (hyphens for readability only):
- Characters 1–12: unique device identifier
- Characters 13–16: make/model code (e.g. C001, A001)
- Characters 17–20: manufacture date in YYMM format

Expiry calculations are accurate to the month. Two-digit year encoding is valid through 2121.

### Alarm index re-use rules

When a linked alarm is removed, its index is freed. New alarms fill the lowest available index. Example: three alarms (indices 0, 1, 2) → remove index 1 → add two new alarms → they take indices 1 and 3.

This confirms that after any alarm removal, the `INDEX→serial` mapping can shift for subsequently added alarms. The `VERIFY→ALARMS→BATTERY` sync loop must be run reliably after any removal or re-pairing event to keep the backend inventory correct. Index drift is a real risk, not just a theoretical one.

### KORE carrier preference

From the KORE Super SIM doc: the OPLMN (Operator PLMN) list forces initial connection to Telstra. If Telstra connection fails, the SIM tries Optus, then Vodafone. Once connected, the SIM will attempt to reconnect to the last-used network first.

This means a hub's first boot in Australia will prefer Telstra. Coverage gaps in Telstra areas will degrade to Optus or Vodafone with some reconnect delay.

### SIM activation thresholds

From the KORE Super SIM doc: a SIM in `ready` state transitions automatically to `active` after any of:
- 250 KB of data consumed
- 5 SMS commands sent or received
- 3 months elapsed

This is relevant for onboarding flows — hubs that are provisioned but not yet installed may auto-activate after 3 months even with no data use.

### CommunicationsQueue feature flag

From the Feature Flags doc: `CommunicationsQueue` — "Enables the overnight communications queue" — is **ON** in Dev, Sandbox, QA, and **Production**.

This flag was unknown during the SMS drop investigation. The overnight queue is a potential pathway for bulk SMS sends. See the SMS drop section below.

### Feature flags approaching archival

Three flags are marked "Ready To Archive":
- `useOdooRCTI` — "Disables RCTI generation inside SaaS." Archiving this means RCTI generation in SaaS is being permanently removed in favour of Odoo handling it.
- `testflag` — original test flag; no active use.
- `audit history` — shows audit history; marked ready to archive October 2023.

The `useOdooRCTI` archival is the operationally significant one: it signals a migration of RCTI generation from the main SaaS API to Odoo.

### Infrastructure hardening gaps

**Incomplete policy:** the remote access session timeout rule reads `(X hours)` — the value was never filled in. This is a formal policy gap that should be closed with a specific timeout value.

**SSH IP whitelist with personal home addresses:** the hardening standards document lists individual SSH access rules by name (e.g. "Hardik Patel Home", "Sam Collins", "Mat Loftus Home") with specific home IP addresses in CIDR /32 notation. These rules have two risks:
1. Home IP addresses change (ISP DHCP rotation), making rules silently stale without anyone updating the policy doc.
2. Individual home addresses in a formal policy document create personnel-linked exposure that is not covered by a VPN or jump box for the listed individuals.

**Security Groups only:** the hardening policy states "Supported network controls for production networks are AWS Security Group ACLs." Network ACLs are not mentioned as an approved control layer. This is consistent with our architecture baseline observation that security groups are the primary boundary control.

---

## Connection to Active Investigations

### SMS Drop Investigation (P1)

The `CommunicationsQueue` feature flag ("Enables the overnight communications queue") is **ON in production** and was not previously known. The P1 SMS drop (93% reduction since March 26–27) has an application-layer root cause with no identified trigger.

A possible hypothesis: the overnight communications queue is the primary batch SMS pathway for routine device status and alert notifications. If the queue processing logic — or anything that feeds it — changed around March 26, the volume drop could follow. The feature flags doc does not show `CommunicationsQueue` being toggled off, but the queue's *content*, *scheduling logic*, or *notification dispatch* could have changed through a code deploy that went to production without a CodeDeploy event (e.g. environment variable change, data change, or logic change in a non-instrumented path).

Recommended: ask the application team to trace exactly what `CommunicationsQueue` processes overnight, whether that processing changed around March 26, and whether it is the primary path for the affected message types.

---

## Summary of Doc Updates Required

| Doc | What to add or change | Priority |
| --- | --- | --- |
| `2026-04-09-alarm-event-traces.md` | Confirm LOW_BATTERY as 3-threshold hub / 1-threshold alarm gap; confirm FAULT=silent discard; add index re-use rules; add HUSH/ALERT STATE encoding (1=On, 2=Off) | High |
| `2026-04-09-mqtt-device-communications.md` | Upgrade QoS mismatch to formally confirmed; add port 18756; correct Twilio→KORE branding; add KORE carrier preference; note FAULT and LOW_BATTERY forward-reference | High |
| `2026-04-11-sms-drop-investigation.md` | Add `CommunicationsQueue` flag as investigation hypothesis | High |
| `docs/README.md` | Add reference to upstream Technical Documents folder and this review note | Normal |
| Firmware spec (upstream) | Flag to team: update "Twilio Super SIM" to "KORE Super SIM" | Low (external doc) |
| Infrastructure Hardening (upstream) | Flag to team: fill in `(X hours)` remote session timeout; review personal home IP SSH rules | Low (external doc) |
| Device Types doc (upstream) | Flag to team: clarify that these are web portal client types, not IoT hardware types; remove or correct VPN claim for browser clients | Low (external doc) |

## Open Questions After This Review

- Does the backend correctly handle `STATE:2` as the "clear/off" signal for `ALERT` and `HUSH`, or does any branch check for `STATE:0`?
- Is `LOW_BATTERY` completely dead in the current backend, or is it handled in a code path not visible in the main MQTT subscriber?
- Is `CommunicationsQueue` the primary pathway for routine notification SMS sends, and did its processing change around March 26?
- Are the personal home IP addresses in the hardening policy still current, or has turnover/ISP churn made them stale?
- Has the `useOdooRCTI` flag been fully archived and the RCTI migration to Odoo completed?
