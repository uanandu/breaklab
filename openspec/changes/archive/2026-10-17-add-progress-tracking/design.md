# Design

## Context

`.breaklab/session.json` (`internal/runner/session.go`) and
`.breaklab/history.json` (`internal/runner/history.go`, change 3) are both
gitignored, machine-local, and scoped to "what's happening right now."
`progress/progress.json` is different in kind: per
`docs/breaklab-spec.md`'s "User data: progress/progress.json ...
(committed)", it's the learner's own durable record, meant to be
committed to their repo like a postmortem. This change adds that file and
the plumbing in `internal/runner` that keeps it current, plus
`breaklab status` to read it back. See `proposal.md` for the full
motivation and `specs/progress/spec.md` for the exact requirement text.

## Goals / Non-Goals

**Goals:**
- Append-only attempt log at `progress/progress.json` matching the shape
  in `docs/breaklab-spec.md`, updated by `Start`/`Check`/`Stop` (not
  `Reset` — see Decision 2).
- `breaklab status`: per-stage completed/total, active lab's elapsed
  minutes and hints used, and a daily streak.

**Non-Goals:**
- Anything postmortem-shaped. `progress.json`'s `postmortem` field and
  the `"complete"` status value are part of the documented shape, so
  this change declares and round-trips them, but nothing here ever
  writes a non-empty `postmortem` or a `"complete"` status — an attempt
  can only reach `active`, `solved`, or `abandoned` until change 6 exists
  to write postmortems and promote `solved` to `complete`.
- The public site (change 7) and web UI (change 8) reading
  `progress.json` — out of scope, though this change's schema is exactly
  what they'll read later.

## Decisions

**1. A new `internal/progress` package, not more code in
`internal/runner`.**
`docs/breaklab-spec.md`'s own package list already names `progress` as
its own concern alongside `catalog`, `docker`, `variants`, and `runner`.
It owns the `Attempt`/`ProgressFile` types, atomic load/save (same
temp-file-plus-rename pattern as `session.go` and `history.go`), the
write operations (`AppendActive`, `MarkSolved`, `MarkAbandoned`,
`IncrementFailedChecks`), and the read-side summary math
(`StageCompletion`, `Streak`) that `breaklab status` needs. `internal/runner`
only calls into it; it never touches the file directly.

**2. `breaklab reset` never touches `progress.json`; it's the same
attempt, not a new one.**
The existing **Reset and stop** requirement already frames `reset` as
"same variant, fresh container" — a retry of one attempt, not the start
of another. `Session` gains `AttemptIndex int` (the array index of the
attempt it belongs to); `Reset` reads the existing `AttemptIndex` off the
session and passes it straight through to the same `launch` helper
`Start` uses, so the rebuilt container's session still points at the
original attempt record. `failed_checks` logged before a reset and after
it both land on that one record, and `started_at` never moves, so
"elapsed minutes" in `breaklab status` reflects the whole struggle,
resets included — not just time since the last rebuild.

**3. `Session.StartedAt` is removed; `progress.json`'s `started_at`
replaces it.**
Before this change, `StartedAt` was written by `launch` on every call —
including `Reset`, which would silently reset it on every retry — and
nothing read it. Once an attempt has its own `started_at` that survives
resets (Decision 2), keeping a second, differently-scoped "started at" on
`Session` is redundant and actively misleading (two timestamps answering
similar-sounding but different questions). Removing it is a small,
directly-motivated cleanup rather than unrelated scope creep: this change
is what makes it redundant in the first place.

**4. Starting a *new* attempt — a different lab, or restarting the
current one without `reset` — first finalizes whatever attempt was
active.**
The spec only states the "stop or switch before solving" case, but
restarting the *same* lab via `breaklab start` (not `reset`) already
silently replaces the session today (it was never required to ask for
confirmation the way switching labs is). Leaving the old attempt's record
stuck on `"active"` forever when that happens would be a real data gap,
so `Start` finalizes the previous attempt (see Decision 6) before
creating a new one, in both cases — only the already-specified
switch-labs path asks for confirmation first; restarting the same lab
does not, matching its existing unconfirmed behavior.

**5. `Check` snapshots `hints_used` into `progress.json` only at the
solved/abandoned transition; while active, `breaklab status` reads hints
live from `Session.HintsUsed`.**
`hints_used` on the *active* attempt's record would otherwise need a
write on every `breaklab hint` call just so `progress.json` stays
current — an extra write for a value `breaklab status` can read straight
from `Session` instead, since the session already carries the live
count. The persisted field only needs to be accurate once the attempt
stops changing (solved or abandoned), which is exactly when `Check` or
`Stop` already writes to the record anyway.

**6. Finalizing an attempt is idempotent: it only overwrites a record
still marked `"active"`.**
`MarkSolved`/`MarkAbandoned`/`IncrementFailedChecks` all no-op if the
target record's status isn't `"active"` any more. This covers two real
cases without special-casing either: running `breaklab check` again
after already solving (shouldn't bump `solved_at` to a later, wrong
time), and `Start`'s "finalize whatever was active" step (Decision 4)
running on an attempt that was *already* solved but never stopped —
that one should stay `"solved"`, not flip to `"abandoned"`, matching the
spec's own "before solving" qualifier.

**7. Streak definition: consecutive local calendar days with at least
one `solved`/`complete` attempt, with one day's grace.**
The spec says "the current daily streak is shown" without defining it.
Timestamps are stored as UTC (`time.Now().UTC()`, matching
`session.json`/`history.json`'s existing convention), but a "day" for a
streak is a human, local-calendar concept, so streak math converts to
local time first. The streak counts backward from today: if today has no
solved attempt yet, that alone doesn't break the streak (the day isn't
over), so counting starts from yesterday instead; from there it counts
consecutive solved days backward until it finds a gap. A learner who
solved labs on three straight days and hasn't solved anything yet today
still sees "3," not "0" — the same grace most habit-streak trackers give.

**8. `progress/progress.json` is world-readable (`0644`), unlike
`session.json`/`history.json`'s `0600`.**
Those two are machine-local run state; this file is a committed artifact
the learner is meant to open, edit, and `git add`, so it gets ordinary
file permissions instead of the session files' "local-only" mode.

## Risks / Trade-offs

- **[Risk]** Removing `Session.StartedAt` (Decision 3) is a breaking
  change to the `Session` JSON shape for anyone with an existing
  `.breaklab/session.json` from before this change. → **Mitigation**:
  that file is gitignored, machine-local, and already disposable by
  design (`breaklab stop` clears it, a corrupt file just needs deleting);
  losing one stale field across an upgrade has no lasting effect.
- **[Trade-off]** Decision 7's "one day's grace" means a learner who
  solves nothing for a full calendar day still sees their old streak
  number right up until the grace day also passes with no activity.
  → **Mitigation**: accepted; it's the standard, expected shape for a
  "streak" and matches what `docs/breaklab-spec.md`'s own inspiration
  (devopslabs.dev-style roadmaps) implies without spelling out.

## Spec clarifications worth flagging

Per the request to call out anything in the spec that reads unclear:

1. **"daily streak" is undefined.** Decision 7 resolves it; flagging in
   case the intent was a stricter definition (e.g. streak breaks the
   instant a day passes with no activity, no grace) — that's a one-line
   change to the streak function if so.
2. **The spec doesn't say what `breaklab reset` or restarting an
   already-active lab should do to attempt records.** Decisions 2 and 4
   resolve both; flagging because change 6 (postmortems) will read
   `progress.json`'s `started_at`/`solved_at` to compute "minutes to fix"
   for its template, and that calculation is only correct if everyone
   agrees `started_at` means "first start of this attempt," not "most
   recent container build."
3. **Whether `failed_checks`/`hints_used` are live or snapshotted isn't
   specified.** Decision 5 resolves it for `hints_used`; `failed_checks`
   is simpler since nothing outside `progress.json` ever needs a live
   count of it, so it's always written straight to the record as it
   happens, with no session-side mirror.
4. **The attempt shape names a `"complete"` status with no requirement
   in this spec ever producing it.** Expected — `"complete"` is reached
   only once change 6 exists to write a postmortem and promote a solved
   attempt. Declaring and validating the shape now means change 6 won't
   need to touch `progress.json`'s struct at all, only add the
   promotion logic.

## Open Questions

None — every ambiguity found had a specific, documented resolution above
rather than a deferred one, since each touches the on-disk shape change
6 and 7 will read.
