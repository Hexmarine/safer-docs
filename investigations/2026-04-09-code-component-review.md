# Code Component Review

- Date: 2026-04-09
- Scope: classify cloned `code/` repositories by likely relevance to the current Sensor Global production platform
- Related systems: Sensor platform runtime, mobile apps, Odoo/CRM, SSO, operational support tooling, upstream architecture documents

## Objective

Review the newly added cloned repositories under `code/`, compare them with the upstream documents and current platform evidence, and identify which repos appear to be in the current production path versus support-only, historical, prototype, or incomplete clones.

## Sources Used

- Repository files:
  - `docs/upstream/SENSOR ARCHITECHTURE DOCUMENT_JAN2025.docx`
  - `docs/upstream/Sensor Global System Description - Rev 6 - July 28 2025.docx`
  - `docs/upstream/Sensor Technical Brief.docx`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
  - `docs/infra/ap-southeast-2-architecture.md`
- Code repositories and files:
  - `code/*` repository remotes, current branches, latest commits, and READMEs
  - key app/config files from `sensor-alarm-backend`, `sensor-angular`, `sso-provider`, `odoo-configuration`, `sim-control`, `pme-shim`, and smaller utility repos
- Local inspection:
  - directory inventory under `code/`
  - extracted text from upstream `.docx` documents
  - text search for PropertyMe, SSO/OIDC, MQTT, Kafka, MongoDB, Odoo, KORE, New Relic, OpenVPN, and environment URLs

## Findings

- The upstream documents and current AWS architecture point to a smaller core platform than the total set of cloned repositories.
- The clearest current platform components are:
  - Node/Express backend API
  - Angular web portals
  - native iOS and Android apps
  - standalone SSO/OIDC provider
  - Odoo CRM/configuration/customization repos
- Several cloned repos are still relevant to operations or debugging, but they do not look like current production runtime components.
- Several smaller repos explicitly describe themselves as prototypes, development helpers, debug tools, or one-off utilities.
- `firmware` is a broken or incomplete local clone at the moment:
  - local repo has no checked-out commits
  - `git status` reports `No commits yet on main...origin/main [gone]`
  - there is no usable source tree to review

## Repo Classification

| Repo | Classification | Evidence | Confidence |
| --- | --- | --- | --- |
| `sensor-alarm-backend` | Direct current platform component | Upstream docs describe a Node.js monolithic API with MQTT, MongoDB, Kafka, MySQL, Twilio/SendGrid style integrations; repo contains Express/TypeScript backend, MQTT service, Kafka producer/consumer, Mongo usage, SSO integration, New Relic, Twilio, LaunchDarkly, and PropertyMe integrations | high |
| `sensor-angular` | Direct current platform component | Upstream docs describe Angular web software; repo contains Angular app with environment URLs for `api`, `admin`, `agent`, `trader`, `owner`, `activation`, `reschedule`, and `crm-*`; matches current multi-environment platform shape | high |
| `smokealarm-android` | Direct current platform component | Upstream docs explicitly say Android app is built natively; repo is active Kotlin Android app clone with recent commits | high |
| `smokealarm-ios` | Direct current platform component | Upstream docs explicitly say iOS app is built natively; repo is active iOS app clone with recent commits and environment setup notes | high |
| `sso-provider` | Direct current platform component | Current AWS inventory shows dedicated SSO API instances and ALBs; backend and frontend both reference SSO/OIDC flows; repo is a standalone OIDC provider with deployment scripts and Mongo backing store | high |
| `odoo-addons` | Direct current platform component | Upstream docs state Odoo 16 Enterprise is used as internal CRM with custom integration points; repo contains many `sg_*` custom addons and is recently active | high |
| `odoo-configuration` | Direct current platform component | Frontend points users to `crm-*` domains and helpdesk/slides URLs; AWS footprint includes Odoo instances; repo contains deployment/configuration for Sensor Global Odoo stack and Caddy domain setup | high |
| `odoo-enterprise` | Direct current platform component | Upstream docs explicitly mention Odoo 16 Enterprise; repo is the enterprise codebase dependency used by the Odoo deployment stack | high |
| `sim-control` | Operational/support repo, not current prod runtime | Upstream technical brief mentions KORE Super SIM; repo manages KORE fleets and SIM activation/deactivation for hardware operations; useful operationally but not part of the main application runtime | high |
| `pme-shim` | Operational/support repo, not current prod runtime | Backend and frontend show active PropertyMe integration; repo README says it is used to download PropertyMe data for debugging; support/debug utility, not primary prod runtime | high |
| `odoo-addons-ci` | Operational/support repo, not current prod runtime | Repo automates addon deployment via cron/CI; supports Odoo delivery workflow but is not itself an end-user runtime component | medium |
| `csv-gen` | Likely historical / prototype / one-off | README describes a local CSV generation utility; no cross-links from main platform repos or upstream docs | medium |
| `mongo-query` | Likely historical / prototype / one-off | README describes ad hoc event report extraction from Mongo; useful for manual reporting/debugging, but no evidence it is part of the deployed platform | medium |
| `mqtt-prototype` | Likely historical / prototype / one-off | README explicitly calls it a controller prototype; useful for protocol testing but not the deployed backend itself | high |
| `odoo-docker-dev` | Likely historical / prototype / one-off | README explicitly says development-only Docker environment for Odoo; local/dev helper, not current production path | high |
| `odoo-installation-script` | Likely historical / prototype / one-off | Generic install/uninstall helper for local Odoo setup; no evidence of current production dependency | medium |
| `repo-tool` | Likely historical / prototype / one-off | Generic Git branch hygiene utility; no platform linkage beyond developer workflow | medium |
| `sensor-cli` | Likely historical / prototype / one-off | Old local CLI with direct DB access and minimal integration evidence; no cross-links from current platform components | medium |
| `testing-secrets-removal` | Likely historical / prototype / one-off | No meaningful product or platform purpose is documented; appears to be a test/sandbox repo | high |
| `wild-dog-solutions-wrapper` | Likely historical / prototype / one-off | README explicitly says wrapper/prototype/proof of concept for external API; no current platform cross-links found | high |
| `firmware` | Broken or incomplete clone state | Local clone contains only `.git`, no checked-out commits, and no source tree to assess | high |

## Summary By Group

### Direct current platform components

- `sensor-alarm-backend`
- `sensor-angular`
- `smokealarm-android`
- `smokealarm-ios`
- `sso-provider`
- `odoo-addons`
- `odoo-configuration`
- `odoo-enterprise`

### Operational/support repos, not current prod runtime

- `sim-control`
- `pme-shim`
- `odoo-addons-ci`

### Likely historical / prototype / one-off

- `csv-gen`
- `mongo-query`
- `mqtt-prototype`
- `odoo-docker-dev`
- `odoo-installation-script`
- `repo-tool`
- `sensor-cli`
- `testing-secrets-removal`
- `wild-dog-solutions-wrapper`

### Broken or incomplete clone state

- `firmware`

## Likely De-Prioritization Candidates

- The safest repos to de-prioritize for current architecture investigation are:
  - `mqtt-prototype`
  - `wild-dog-solutions-wrapper`
  - `testing-secrets-removal`
  - `csv-gen`
  - `repo-tool`
  - `odoo-docker-dev`
  - `odoo-installation-script`
- These may still have historical or developer value, but they do not currently look necessary for understanding the production AWS platform.

## Important Caveats

- `Not in current production path` is not the same as `unused anywhere`.
- `sim-control` and `pme-shim` look genuinely useful, but primarily for operations and debugging rather than runtime service delivery.
- `firmware` should not be interpreted as unused firmware overall. It only means the local clone is incomplete and cannot yet be reviewed.
- Commit recency helped with confidence, but code/config linkage and upstream architecture evidence were treated as stronger signals than date alone.

## Recommended Next Steps

- If the goal is platform architecture mapping, focus further review effort on:
  - `sensor-alarm-backend`
  - `sensor-angular`
  - `sso-provider`
  - `odoo-*`
  - mobile repos only as needed for client-facing behavior
- If the goal is repo cleanup or archiving, first validate with owners whether `sim-control`, `pme-shim`, and `odoo-addons-ci` are still part of active operational workflows.
- Re-clone or repair `firmware` before making any judgment about its business relevance.

## Open Questions

- Which team currently owns the SSO provider and Odoo deployment path?
- Are `sensor-cli` and `mongo-query` still used informally by support or engineering staff?
- Is there a current firmware repository state that failed to clone, or has firmware work moved elsewhere?
