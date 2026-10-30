## Why

Once a lab is marked complete it is never revisited. Incident-response skill decays with disuse the same way any hands-on skill does, but nothing in BreakLab ever resurfaces a solved lab to check whether the user could still solve it cold. The roadmap and challenge browser both treat "complete" as a permanent, final state, so the one signal that would actually indicate durable skill — can you still do this without hints, months later — is never collected.

## What Changes

- A completed lab becomes "due for review" once enough time has passed since its most recent complete attempt's `solved_at` (a fixed interval, e.g. 60 days — exact value decided in design.md).
- The roadmap and challenge browser surface a "Review due" badge on a due lab, visually distinct from `available`/`in-progress`/`complete`, without changing what `status` a lab reports today (review-due is an annotation, not a new status value other code branches on).
- Starting a review attempt re-runs the lab exactly like a normal attempt (same container, same `check.sh`), but the CLI/web UI withhold hints for the duration of a review attempt — the whole point is confirming unaided recall, not re-practicing with help.
- A completed review attempt is recorded as a normal attempt in `progress/progress.json` (no new persisted field); its `solved_at` becomes the new baseline the next due date is computed from.
- Explicitly out of scope: spaced-repetition scheduling algorithms (e.g. SM-2) — a single fixed interval is enough to start; per-lab custom intervals; notifications/reminders outside the existing web UI and CLI `status` output.

## Capabilities

### New Capabilities
- `lab-retention-review`: computing whether a completed lab is due for review, withholding hints during a review attempt, and surfacing the due state in `breaklab status`, the roadmap, and the challenge browser.

### Modified Capabilities
(none — review-due is computed from the existing `progress` capability's `solved_at` field, no schema change)

## Impact

- `internal/progress`: new pure function computing review-due state from existing attempt history (no new persisted field).
- `internal/runner`: hint command checks review-due state for the active attempt and refuses/withholds hints during a review attempt.
- `internal/web`: `roadmap.go`/`challenges.go` annotate `LabRowView` with a `ReviewDue` flag; `roadmap.html`/`challenges.html` render the badge.
- `cmd/breaklab`: `status` subcommand lists labs due for review alongside existing stage completion counts.
