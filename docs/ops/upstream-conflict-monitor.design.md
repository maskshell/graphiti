# Design — upstream conflict monitor (GitHub Actions)

Status: draft for adversarial review (solidforge pipeline: bc -> csr -> pd).

## Problem

PR #1209 lives on fork branch `main`. Upstream `getzep/graphiti` moves actively (3 merges in 2 days at time of writing). Every upstream advance risks (a) GitHub's "out-of-date" flag and (b) real conflicts. The contributor needs timely notification of conflicts without watching the PR page, including when no local machine is online.

## Constraints (hard)

1. C1 — Zero PR pollution: no file this automation needs may appear in the PR branch's diff. Fork default branch == PR branch (`main`), and GitHub scheduled workflows run ONLY from the default branch. Therefore the workflow must live on a dedicated branch and the fork's default branch must be switched to it.
2. C2 — Notification must reach the owner without any local agent: GitHub issue creation on the fork notifies the repo owner by email.
3. C3 — No secrets: the workflow must operate with the default `GITHUB_TOKEN` only (scope `issues: write`), never a PAT.
4. C4 — Idempotent alerting: repeated conflict states must update/continue ONE tracking issue, never spam new issues; resolved conflicts must auto-close that issue.
5. C5 — Must not run on upstream's copy if this branch is ever merged there: guard `if: github.repository == 'maskshell/graphiti'`.
6. C6 — Known platform limit (disclose, not solve): GitHub deactivates scheduled workflows after 60 days of repo inactivity. Any push to the fork re-arms it. The design treats >60-day total silence as out of scope.

## Requirements

- R1: Hourly check (cron offset from :00 to reduce schedule-slot contention).
- R2: Three-state classification of (upstream/main vs fork main): `contained` (no action), `clean-behind` (rebase advisable; non-alerting log only), `conflict` (alert).
- R3: On conflict: create-or-update tracking issue listing conflicting files and both heads; link the Actions run.
- R4: On transition conflict -> non-conflict: close the tracking issue with a resolution note.
- R5: Manual `workflow_dispatch` trigger for on-demand runs (also serves as the 60-day re-arm test).
- R6: Detection must not mutate any branch: `git merge-tree --write-tree` (no worktree side effects), never a real merge/push.

## Detection contract

```
UP   = rev-parse getzep/graphiti@main          (fetched as remote 'upstream')
TIP  = rev-parse origin@main                    (the PR branch)
BASE = merge-base UP TIP
UP == BASE            -> contained
merge-tree TIP UP ok  -> clean-behind
otherwise             -> conflict   (+ file list via --name-only tail)
```

## Alert contract (issue lifecycle)

- Title (fixed, searchable): `ops: upstream/main has conflicts with main (PR #1209)`.
- Conflict & existing open issue -> comment with new heads + files + run link (dedup by title search).
- Conflict & no open issue -> create with body above.
- Non-conflict & open issue -> close with `rebased/resolved at <run>` comment.
- Non-conflict & no issue -> no-op.

## Failure modes & responses

| Failure | Response |
|---|---|
| `git fetch upstream` fails (network) | Job fails loudly (red run) — visible in Actions tab; no false "contained". Absence-of-run is itself a signal. |
| `gh issue` API fails | Job fails; conflict state re-alerts next hour (acceptable duplicate-free retry). |
| Schedule deactivated (60-day rule) | R5 dispatch re-arms; disclosure in C6. |
| Fork Actions disabled | Detectable via absent runs; documented re-enable command in run book. |
| Default branch changed back by owner | Workflow stops scheduling silently — mitigation: this doc records the dependency; check listed in run book. |

## Decisions

- D1: Default-branch switch to `ops/monitor` (vs. repository_dispatch from another repo): keeps everything inside the fork, zero external deps; cost is cosmetic (fork landing page shows ops branch).
- D2: Issue-as-notification (vs. email/webhook): zero secret, native owner email, durable thread history per C2/C3.
- D3: `merge-tree` (vs. checkout+merge attempt): R6, no mutation, single-process.
- D4: `clean-behind` does NOT alert (log only): out-of-date flag is visible on the PR page itself; alerting on it duplicates GitHub's own signal. Conflict is the state needing action.

## Run book

- Manual run: `gh workflow run upstream-conflict-monitor.yml --repo maskshell/graphiti`.
- Re-enable Actions: `gh api -X PUT repos/maskshell/graphiti/actions/permissions -F enabled=true -F allowed_actions=all`.
- Default branch must remain `ops/monitor` for the schedule to fire.

## Open decision points (for review)

- ODP1: Should `clean-behind` also comment on the tracking issue (non-closing) so the owner sees drift even without conflicts? Current: no (D4).
- ODP2: Should the cron be hourly, or 4x daily (reduced Actions minutes)? Current: hourly (matches original ask).
- ODP3: Should the workflow ALSO verify the PR's CI status (checks green) and include it in the issue body? Current: no (scope creep).
