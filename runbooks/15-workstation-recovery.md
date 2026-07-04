# 15 — Dev workstation (Claude workspace) recovery

**Purpose:** restore the development/operations workstation — including the Claude
Code knowledge base — if the instance is lost, corrupted, or must be rebuilt.

**The box (as of 2026-07-03):** `i-0a82d062d3c2976ee` (`yevgen-devbox`,
`StopWhenIdle=true`), root volume `vol-02e6411852fc189d2` (150 GiB gp3,
**encrypted**), user `ubuntu` (migrated from `yevgen`; `~/.claude/projects/` keeps
`-home-ubuntu-*` → `-home-yevgen-*` symlinks for continuity).

---

## Durability map — what lives where

| Data | Location on box | Off-box copy | Notes |
|---|---|---|---|
| Application/code repos | `~/dev/sensorsyn/code/*` | GitHub (SensorGlobal / SaferHomesAu orgs) | ~24 nested repos; clean clones |
| Platform docs, runbooks, investigations | `~/dev/sensorsyn/docs/` | `SaferHomesAu/sensor-safer-docs-investigations` | **Only as fresh as the last push** — commit/push at wrap-up |
| Claude memory, plans, config | `~/.claude/` | Private repo (whitelist `.gitignore`; init 2026-07-03) | Includes all `projects/*/memory/`, `plans/`, CLAUDE.md, settings, keybindings |
| Operational scripts (MFA login, tunnels, diag, import guards, key rotation) | `~/dev/sensorsyn/scripts/` | restic/B2 (6-hourly) | git-untracked — see "Open gaps" below |
| Undo snapshots + PII extracts | `~/dev/sensorsyn/ops-and-extracts/` | restic/B2 (6-hourly) + EBS snapshot | must NOT go to GitHub — PII; restic is client-side encrypted |
| AWS credentials/profile | `~/.aws/` (`sensorsyn-mfa` profile) | restic/B2 (encrypted) | never in git; MFA device re-pair still manual |
| kubeconfig | `~/.kube/config` | restic/B2; or regenerate: `aws eks update-kubeconfig --name safer-ops-prod` | |
| GitHub auth (gh CLI / SSH key) | `~/.ssh`, gh config | restic/B2 (`~/.ssh`); gh re-auth manual | |
| Claude Code auth | `~/.claude/.credentials.json` | `claude login` re-auth | Deliberately excluded from the repo |
| Toolchain (claude, codex via volta, boto3, kubectl, helm, docker, terraform, Playwright chromium) | various | Reinstall (Path B) | |
| Session transcripts (~580 MB) | `~/.claude/projects/*/**.jsonl` | restic/B2 (inside `~/.claude`) | cheap to keep — included |

---

## Path A — restore from EBS snapshot (preferred, ~15 min)

Precondition: the volume is snapshot-covered (see "Open gaps" — pending approval).

1. Find the newest snapshot of `vol-02e6411852fc189d2` (or by `Name=yevgen-devbox`).
2. Create a volume from it (same AZ as the replacement instance) — or launch a new
   instance from an AMI built off the snapshot.
3. Attach as root (or mount as secondary and copy `/home/ubuntu` onto a fresh box).
4. Re-auth everything token-based (all likely stale): `bash scripts/aws-mfa-login.sh`,
   `claude` (re-login if prompted), `gh auth status`.
5. Verify: `claude` starts and recalls memory; `git -C ~/dev/sensorsyn/code/safer-ops status`;
   `AWS_PROFILE=sensorsyn-mfa aws sts get-caller-identity`.

Losses: nothing except changes since the last daily snapshot (09:00 UTC).

## Path A2 — restore from the restic/B2 backup (portable — works on ANY Linux/macOS box)

The off-AWS layer (added 2026-07-04): restic repository `b2:<bucket>:workstation` on
Backblaze B2, client-side encrypted. Covers the non-git residue: `~/.claude` (memory,
plans, config; caches excluded), `~/.claude.json`, `~/.aws`, `~/.ssh`, `~/.kube`,
`~/.zshrc`, `~/.gitconfig`, workspace `scripts/`, `ops-and-extracts/`, `docs/`.
Runs 6-hourly via the `ubuntu` crontab (`scripts/backup-workstation.sh`; log at
`~/.config/restic-workstation/backup.log`; retention 7d/4w/6m).

Needs (all from the password manager / B2 console — nothing from the dead box):
the B2 bucket name + an application key, and the **restic repository password**
(without it the backups are unrecoverable — no reset exists).

```bash
# on the new machine
apt/brew install restic
export RESTIC_REPOSITORY="b2:<bucket>:workstation"
export B2_ACCOUNT_ID="<keyID>" B2_ACCOUNT_KEY="<applicationKey>"
restic snapshots                       # prompts for the repo password; lists history
restic restore latest --target /      # or --target /tmp/rest + selective copy
```

Then re-auth as in Path A step 4 (MFA session, `claude login` if needed, `gh auth`).
Restore round-trip was verified 2026-07-04 (single-file restore, byte-identical).

## Path B — rebuild from scratch (no snapshot, ~half a day)

1. **Launch** Ubuntu LTS instance in `ap-southeast-2` (≥100 GiB gp3, encrypted;
   tag `Name`, `Purpose=devbox`, `StopWhenIdle` as desired).
2. **Toolchain:** git, gh, zsh, python3 + boto3, AWS CLI v2 (beware the
   v2.31/Python 3.14 `send-command` argparse bug — prefer boto3 for SSM), volta →
   node + `@openai/codex`, kubectl, helm, docker, terraform, Claude Code.
3. **Restore the Claude knowledge base:** clone the private `~/.claude` repo to a
   temp dir, copy its contents into a fresh `~/.claude/` (it is a whitelist repo —
   the clone IS the durable subset), then `claude login`. Memory, plans, settings,
   keybindings are back. Re-add the Playwright MCP to `~/.claude.json` per the
   `playwright-mcp-headless-setup` memory (needs `--executable-path` to the
   ms-playwright chromium + `--output-dir`).
4. **Recreate the workspace:** `mkdir ~/dev/sensorsyn && cd ~/dev/sensorsyn`;
   clone `SaferHomesAu/sensor-safer-docs-investigations` as `docs/`; clone the
   needed `code/*` repos (start with sensor-alarm-backend, safer-ops, sso-provider,
   sensor-mcp; the rest on demand). Restore `scripts/` from its off-box copy
   (see gaps) or from the docs of record.
5. **Credentials:** recreate `~/.aws` (IAM user keys + `sensorsyn-mfa` profile,
   MFA device), `aws eks update-kubeconfig --name safer-ops-prod`, `gh auth login`
   + SSH key registered with both GitHub orgs.
6. **Verify** as in Path A step 5.

Losses without a snapshot: `ops-and-extracts/` (incident undo files, PII extracts),
`scripts/` (unless the gap below is closed), shell config, transcripts.

---

## Backup discipline (steady state)

- **Wrap-up habit:** commit/push `docs/` and the `~/.claude` repo at the end of any
  session that changed them. Repos are the *selective/reviewable* layer.
- **restic → B2, 6-hourly cron** is the *portable* layer (restores on any Linux —
  Path A2). Watch `~/.config/restic-workstation/backup.log` if in doubt; run
  `restic check` occasionally.
- **EBS daily snapshot** is the *whole-box* layer covering everything git can't
  hold (PII extracts, creds-adjacent config, toolchain). Retention must stay
  bounded (7-day rolling) — do not recreate the AMI/snapshot sprawl cleaned up
  2026-06-27.

## Open gaps (as of 2026-07-03)

1. ~~No EBS snapshot exists for this volume.~~ **CLOSED 2026-07-04:** added
   `{'Name': 'yevgen-devbox'}` to the `TargetTags` of DLM policy
   `policy-0441001e0e7116959` ("Seven Days Backup", daily 09:00 UTC, 7-day
   age-based retention; policy targets INSTANCE with boot volume included).
   See `docs/applied-changes.md`. Undo = remove that TargetTags entry.
   Confirm the first snapshot exists after the next 09:00 UTC run.
2. **`scripts/` is git-untracked.** Covered by restic/B2 since 2026-07-04, so no
   longer a durability gap — but a git home (own private repo, or fold into the
   docs repo after a secret-scan) would still add history/review.
3. `backups/incident-20260604/` (Haven-incident PITR extracts referenced in
   investigation notes) **no longer exists on disk** — treat as gone; the RDS PITR
   window has long since passed. Job-closure undo CSVs still exist under
   `ops-and-extracts/`.
