## Context

`internal/progress` already records every attempt (lab, variant, started_at, solved_at, status) and `internal/progress.LatestCompleteAttempts` (used by `internal/web/roadmap.go` and `challenges.go` today) already finds each lab's most recent complete attempt. `internal/runner`'s hint command already knows the active attempt's lab and variant. No new persisted state is required — see proposal.md - Capabilities.

## Goals / Non-Goals

**Goals:**
- Review-due is a derived, point-in-time computation, never a stored flag that can drift from the attempt log it's computed from.
- A review attempt reuses every existing mechanic (container lifecycle, `check.sh`, attempt recording) — only hint availability differs.

**Non-Goals:**
- No spaced-repetition scheduling (SM-2 or similar) — see proposal.md.
- No new progress.json fields or migration of existing progress.json files.

## Decisions

**Review interval is a package constant, not configuration.** A single `const reviewInterval = 60 * 24 * time.Hour` in `internal/progress` (exact value: 60 days — long enough that genuine retention, not short-term memory, is what's being tested). Alternative considered: a per-lab or user-configurable interval — rejected for v1 as unneeded complexity; nothing in the proposal calls for different labs decaying at different rates.

**Review-due is computed, not stored.** A new `progress.ReviewDue(cat *catalog.Catalog, pf *ProgressFile) map[string]bool` (or similar), built the same way `LatestCompleteAttempts` already is, called from the same three call sites that call `ApplyStatus`/`LatestCompleteAttempts` today (`roadmap.go`, `challenges.go`, the `status` command). Alternative considered: persist a `review_due` boolean on each attempt, recomputed and written back periodically — rejected because it introduces a cache that must be invalidated correctly, for a value cheap enough to recompute on every read.

**A review attempt is identified by "lab already complete + attempt just started," not a new attempt field.** When `runner.Start` begins an attempt for a lab that already has a complete attempt in progress.json, the resulting session is a review attempt. The hint command checks this the same way at request time — no new session or progress.json field. Alternative considered: a `--review` flag on `breaklab start` the user must remember to pass — rejected because it relies on the user self-reporting the exact condition the system can already detect.

**Hints are refused outright, not just hidden in the UI.** The CLI's `hint` command and the web UI's hint-reveal handler both check review-attempt status server-side before revealing anything, so there's no path (including a direct API call) that leaks a hint during a review attempt.

## Risks / Trade-offs

[A 60-day interval may feel arbitrary and wrong for some labs] → Not configurable in v1; revisit with real usage data rather than guessing a per-lab schedule upfront.

[Computing review-due on every page/status render re-scans every lab's attempts] → Already the exact cost `LatestCompleteAttempts` pays today for the same data; no new scan, just one more derived field per lab alongside the existing status computation.

## Migration Plan

No data migration: existing progress.json files already contain every field this feature reads. Ship behind no flag — review-due badges simply start appearing for any lab whose existing `solved_at` already predates the interval.
