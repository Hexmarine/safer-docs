# SensorSyn platform docs

Working knowledge for the SensorGlobal → Safer Homes platform handover and
migration: discovery notes, current-state infrastructure reference, executable
runbooks, and the open-questions/gaps backlog.

## Map

| Where | What |
|---|---|
| [`investigations/`](investigations/README.md) | **Dated discovery notes** — architecture, cost/security analysis, incident root-causes, migration planning. Start at its index. |
| [`infra/`](infra/) | **Stable topic reference** — current-state architecture, ownership, pipelines, access. Updated in place as things change. |
| [`runbooks/`](runbooks/README.md) | **Executable procedures** — numbered, with baseline → approval gate → verification → rollback. |
| [`gaps.md`](gaps.md) / [`gaps-detailed.md`](gaps-detailed.md) | Open access/ownership/operational **gaps** — compact list + full context tables. |
| [`questions.md`](questions.md) | Open product-walkthrough questions (resolved + outstanding). |
| [`applied-changes.md`](applied-changes.md) | Append-only log of approved changes actually executed. |
| [`upstream/`](upstream/) | Vendor + internal reference docs (incl. 12 technical PDFs; firmware spec, event types, install process, …). |
| [`templates/`](templates/) | Reusable templates for new investigation notes. |

The depot **prepared-kit install flow** built on top of this platform lives in its
own repo and docs: [`../code/safer-ops/docs/`](../code/safer-ops/docs/README.md).

## Start here

- **Who owns what / JV context:** `infra/company-and-ownership.md`
- **Repo layout, build commands, AWS access:** `infra/repo-operations-reference.md`
- **Current prod architecture:** `infra/ap-southeast-2-architecture.md`
- **What we still need (access + answers):** `gaps.md`
- **Platform & device data flow:** `investigations/2026-04-09-platform-architecture-and-data-flow.md`
- **Migration plan + account detach:** `investigations/2026-04-10-migration-plan.md`, `runbooks/09-aws-account-standalone-detach.md`
- **New install flow durability plan:** `investigations/2026-05-20-installation-flow-durability-plan.md`

## Other top-level trackers

- `low-traffic-optimisation-backlog.md` — quiet-month cost/capacity execution tracker
- `local-k8s-validation-backlog.md` — local Kubernetes/MQTT validation backlog
- `management-cost-summary.md` — historical cost-proposal summary (pre-decommissioning; see the 2026-04-20 inventory + cost-optimization-proposals for current figures)
- `sso-client-registration.md` — safer-ops SSO client creation research

## Working rules

- Prefer a **new dated note** per meaningful investigation; don't rewrite old ones — they're point-in-time records.
- Investigation notes hold findings, hypotheses, recommendations, open questions.
- `applied-changes.md` records only changes actually executed and confirmed.
- Keep `infra/` topic docs factual and current; link back to the investigation that produced them.
- When a finding graduates into stable reference or a procedure, move it to `infra/` or `runbooks/` and leave the dated note as history.

## Naming

`investigations/YYYY-MM-DD-short-topic.md` — e.g. `investigations/2026-04-09-aws-network-overview.md`
