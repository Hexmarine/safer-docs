# Optimisation Ops Repo Bootstrap

This note defines the recommended shape for a separate repository dedicated to **executing**
approved cost optimisations and tracking what was actually applied.

## Purpose

Use the separate repo for:
- approved runbooks
- reusable execution scripts
- applied-changes tracking
- approval request templates
- small inventory / allowlist files used by automation

Do **not** use the separate repo for:
- exploratory investigations
- architecture discovery notes
- deep-dive evidence collection
- upstream reference material
- copied application source code

Those stay in this repository.

## Why split

The current repository is a mixed investigation workspace. A dedicated ops repo is cleaner once the
work shifts from discovery into repeated execution.

Benefits:
- cleaner PR trail for operational changes
- smaller review surface for execution scripts
- easier ownership boundaries
- simpler `applied-changes.md` history
- less noise from large investigation notes

## Recommended repo name

Examples:
- `sensorsyn-ops-optimisation`
- `sensorsyn-cost-ops`
- `sensorsyn-infra-operations`

## Recommended layout

```text
README.md
docs/
  applied-changes.md
  runbooks/
  approvals/
  inventory/
templates/
  approval-request.md
  applied-change-entry.md
scripts/
  health-check.sh
  set-log-retention.sh
  stop-nonprod.sh
  start-nonprod.sh
```

## Files to seed from this repo

### Copy first

| Current file | New repo path | Why |
|---|---|---|
| `docs/applied-changes.md` | `docs/applied-changes.md` | canonical executed-change log |
| `docs/runbooks/README.md` | `docs/runbooks/README.md` | runbook usage index |
| `docs/runbooks/03-log-retention.md` | `docs/runbooks/03-log-retention.md` | first approved automation candidate |
| `docs/runbooks/08-stop-nonprod-environments.md` | `docs/runbooks/08-stop-nonprod-environments.md` | high-value non-prod cost control |
| `scripts/health-check.sh` | `scripts/health-check.sh` | baseline + verification harness |
| `scripts/set-log-retention.sh` | `scripts/set-log-retention.sh` | retention automation |
| `scripts/stop-nonprod.sh` | `scripts/stop-nonprod.sh` | non-prod shutdown automation |
| `scripts/start-nonprod.sh` | `scripts/start-nonprod.sh` | non-prod resume automation |

### Reference only, do not copy by default

| Current file | Why reference instead of copy |
|---|---|
| `docs/investigations/2026-04-11-cost-optimization-proposals.md` | source evidence and proposal rationale; better linked than duplicated |
| `docs/management-cost-summary.md` | management-facing summary still belongs with broader investigation set |
| dated investigation notes under `docs/investigations/` | historical evidence, not execution assets |
| `docs/upstream/` and other reference docs | source material, not operational content |

## Workflow model

1. Investigation happens in this repo
2. Once an optimisation is approved and execution-ready, copy the relevant script/runbook into the
   ops repo
3. Execute from the ops repo
4. Record the actual change in `docs/applied-changes.md`
5. Link back to the evidence note in this repo

## Suggested README sections for the new repo

1. Purpose and scope
2. Approval rules
3. Execution rules
4. Change logging rules
5. Script inventory
6. Runbook index
7. Links back to investigation evidence in this repo

## Suggested execution rules

- No remote change without written approval reference
- Every applied change must append an entry to `docs/applied-changes.md`
- Scripts default to `--dry-run` where possible
- Production changes require explicit scope and rollback notes
- Investigation notes do not belong in the ops repo

## First bootstrap step

If you create the new repo now, seed it only with:
- `docs/applied-changes.md`
- `docs/runbooks/README.md`
- `docs/runbooks/03-log-retention.md`
- `docs/runbooks/08-stop-nonprod-environments.md`
- `scripts/health-check.sh`
- `scripts/set-log-retention.sh`
- `scripts/stop-nonprod.sh`
- `scripts/start-nonprod.sh`

That keeps the first version small and execution-focused.
