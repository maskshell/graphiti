# Design — upstream conflict monitor (GitHub Actions)

Status: converged v2 (bc → csr same-family + different-family → pd). Supersedes v1.

## Problem

PR #1209 lives on fork branch `main`. Upstream `getzep/graphiti` moves actively. Every advance risks GitHub's out-of-date flag and real conflicts. The owner needs timely conflict notification with no local agent online.

## Constraints (hard)

1. C1 — Zero PR pollution: no file may appear in the PR branch diff. The workflow lives on `ops/monitor`; the fork default branch is switched to it (scheduled workflows run only from the default branch).
2. C2 — Notification reaches the owner setting-independently: the issue body @mentions the owner (`cc @maskshell`). Per GitHub notification docs, auto-watch excludes forks and email delivery is user-configurable, so bare issue creation is NOT a reliable channel; @mention notification is.
3. C3 — No secrets: default `GITHUB_TOKEN`, scope `issues: write` (+ `contents: read`), `GH_REPO` set explicitly on every gh call.
4. C4 — Idempotent alerting: exactly ONE tracking issue, keyed by a repo-level label `upstream-conflict` (repo metadata, not a git file — C1-safe). Label idempotent-create on each run. Repeated conflict states comment only when the state fingerprint changes. Manual close is respected (see A5).
5. C5 — Never runs on upstream: `if: github.repository == 'maskshell/graphiti'`.
6. C6 — Platform limit (corrected per GitHub docs): in a PUBLIC repository, scheduled workflows are automatically disabled when no repository activity occurred in 60 days. Reactivation is documented as `gh workflow enable` (or Actions UI "Enable workflow"), or a commit by a write-permission user that CHANGES the cron schedule. A plain push does NOT re-arm a disabled workflow. Out of scope beyond the run-book step.
7. C7 — Run serialization: `concurrency: { group: upstream-conflict-monitor, cancel-in-progress: true }` — schedule × dispatch collisions cannot double-create the tracking issue.

## Requirements

- R1: Hourly cron, offset from :00 (GitHub delays/drops runs at high load, notably top-of-hour; a delayed or dropped run is subsumed by the next hourly run and does not change issue state).
- R2: Four-state classification: `contained`, `clean-behind`, `conflict`, `error` (detection failure). The check step ALWAYS exits 0 and emits `status` + `conflict_files` + heads as outputs; a red run means infrastructure failure ONLY (fetch/gh errors).
- R3: On `conflict` with no OPEN labeled issue: create it (files, both heads, run link, @mention owner).
- R4: On non-conflict (`contained`/`clean-behind`) with an open labeled issue: close it with a resolution comment naming the state and run. (This implements the conflict→resolved transition.)
- R5: `workflow_dispatch` for on-demand runs and deploy verification. It is NOT a re-arm path for a disabled workflow (see C6).
- R6: Detection never mutates branches: `git merge-tree --write-tree --name-only` between the two fetched heads. Conflict file list = lines after the tree OID up to the first blank line (`sed -n '2,/^$/p'`); v1's `/^CONFLICT/` capture grabbed message lines, not files.
- R7: Fingerprint-gated comments: comment on the open issue only when (upstream head, conflict file set) changed since the last comment; unchanged conflict = silent success (prevents ~24 comments/day).

## Alert contract (issue lifecycle)

| State | Open labeled issue | Action |
|---|---|---|
| conflict | none | create (R3) |
| conflict | open, new fingerprint | comment (R7) |
| conflict | open, same fingerprint | no-op |
| conflict | closed (manually) | no-op — manual close is respected; owner has seen it |
| contained / clean-behind | open | close with resolution note (R4) |
| contained / clean-behind | none/closed | no-op |
| error | any | no alert-path action; red run is the signal (visible in Actions tab only — see Failure modes) |

## Failure modes & responses

| Failure | Response |
|---|---|
| fetch upstream fails | check step exits nonzero → red run. No false contained, no bogus issue. NOTE: absence-of-run/red-run has NO push notification channel by design; the owner observes the Actions tab (accepted limitation, per docs notifications go to the cron-syntax modifier only for runs that happen). |
| gh issue/label API fails | red run; state retried next hour, duplicate-free (label dedup) |
| schedule delayed/dropped at peak | next hourly run subsumes; no state change |
| workflow disabled (60-day rule) | re-arm per C6: `gh workflow enable upstream-conflict-monitor.yml` |
| fork Actions disabled (repo level) | `gh api -X PUT repos/maskshell/graphiti/actions/permissions -F enabled=true -F allowed_actions=all` then workflow-level enable |
| default branch reverted by owner | schedule stops silently; run book records the dependency |
| no merge-base (unreachable in practice) | detection step fails red under `set -eu` → surfaces as error, not as false contained |

## Decisions (delta from v1 review)

- D5: `if: always()` reconciliation step consumes check outputs (W1) — red runs mean infra only.
- D6: label-based dedup replaces title search (W3).
- D7: @mention in issue body (W4 / different-source C2 finding).
- D8: fingerprint-gated comments (W6).
- D9: respect manual close (W5).
- D10: inherited upstream scheduled workflows on the fork (CodeQL, stale, intake bots) are disabled at the FORK WORKFLOW LEVEL (UI-equivalent, zero file edits, C1-safe) to keep the Actions signal clean.
- ODP1: NO (keep D4 — clean-behind closes, never comments). ODP2: KEEP hourly (public-repo scheduled runs are free; detection latency is the only cost). ODP3: NO (CI status reaches the owner via native PR notifications; folding it in couples the monitor to lint flakiness).

## Run book

- Manual run: `gh workflow run upstream-conflict-monitor.yml --repo maskshell/graphiti`.
- Re-arm after 60-day deactivation: `gh workflow enable upstream-conflict-monitor.yml --repo maskshell/graphiti` (C6).
- Default branch must remain `ops/monitor`.
- Disabled inherited schedules: CodeQL Advanced, Issue intake, AI Moderator, Close stale incomplete items (D10).

## Retirement (post-merge)

After PR #1209 merges: revert the fork default branch to `main` (or delete the fork), delete `ops/monitor`, close any open tracking issue. Without this, `git fetch origin main` fails hourly forever.
