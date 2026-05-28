# Repo Governance Bootstrap

- Date: 2026-04-09
- Scope: repository guardrails and documentation structure for AWS investigation work
- Related systems: repository process, AWS discovery workflow, documentation standards

## Objective

Establish safe operating rules for a Codex-driven workflow around an existing AWS environment with production risk.

## Sources Used

- Repository files: existing `docs/` directory layout
- AWS commands or consoles: none
- External references: none

## Findings

- The repository needed a clear agent policy to prevent unapproved infrastructure changes.
- Documentation needed both an index and durable note structure so investigations stay organized over time.
- A separate applied changes log is useful only for confirmed approved changes, not for proposals.

## Risks Or Constraints

- The AWS environment is assumed production-impacting unless a human states otherwise.
- Unsafe automation is more likely when helper scripts mask mutating behavior, so the guardrails need to prohibit both direct and wrapped infra mutations.
- Infra-related repo edits can themselves become risky if they are made without explicit written approval.

## Cost Optimization Opportunities

- No environment-specific opportunities identified yet.
- The process now creates a dedicated place to record future savings investigations and supporting evidence.

## Recommended Next Steps

- Keep future discovery work in dated investigation notes.
- Add stable current-state summaries under `docs/infra/` as architecture knowledge becomes reliable.
- Require explicit written approval before any edit to IaC, deployment scripts, or prod-impacting config.

## Open Questions

- Which AWS accounts, regions, and environments are in scope for regular investigation?
- Which repo files should be classified as infra-related once they exist?
