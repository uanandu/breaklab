# Proposal

## Why

Nothing today remembers what a learner has actually solved, or for how
long they struggled. This is change 5 of 9 in the v1 build plan: it adds
`progress/progress.json` — a committed, append-only attempt log — and
`breaklab status`, so the learner can see per-stage completion, the
active lab's elapsed time and hints used, and a daily streak, all read
live from files they own.

## What Changes

- New `internal/progress` package: the `Attempt`/`ProgressFile` types
  matching the `progress.json` shape in `docs/breaklab-spec.md`, atomic
  load/save, and the operations the runner needs to keep it current
  (start a new attempt, mark solved, mark abandoned, count a failed
  check) plus the read-side summary `breaklab status` needs (per-stage
  completion, streak).
- `progress/progress.json` is created on first use and is **not**
  gitignored — unlike `.breaklab/session.json` and `.breaklab/history.json`,
  this is the learner's own committed record, per
  `docs/breaklab-spec.md`'s "User data: progress/progress.json ...
  (committed)".
- `Session` gains `AttemptIndex int`, so each command's own process
  knows which `progress.json` entry belongs to the running attempt.
  `Session.StartedAt` is removed: it tracked "when was this container
  last (re)built," which `progress.json`'s own `started_at` (set once per
  attempt, preserved across resets) now makes redundant — see design.md.
- `Runner.Start`, `.Check`, `.Reset`, and `.Stop` are wired to keep
  `progress.json` current: a new attempt record per `Start` (abandoning
  any still-active prior attempt first), a failed-check count per failing
  `Check`, a "solved" transition on a passing `Check`, an "abandoned"
  transition on `Stop` (only if not already solved), and no record
  mutation at all from `Reset` (same attempt, same index).
- New `breaklab status` command: each roadmap stage shows completed labs
  out of total; the active lab (if any) shows elapsed minutes and hints
  used; the current daily streak is shown.
- Out of scope: postmortems (`progress.json`'s `postmortem` field and the
  `"complete"` status value are declared per the shape in
  `docs/breaklab-spec.md` but nothing in this change ever sets either —
  that's change 6), the public progress site (change 7), and the web UI
  (change 8).

## Capabilities

### New Capabilities

- `progress`: attempt logging to `progress/progress.json` (start, solve,
  abandon, failed-check counting) and the `breaklab status` summary
  (per-stage completion, active-lab elapsed/hints, daily streak).

### Modified Capabilities

_None._ `lab-runner`'s existing requirements (starting, checking,
resetting, stopping) keep their documented behavior; this change adds
progress bookkeeping alongside them without changing what any of those
commands show the user or how they decide pass/fail, so no requirement
text in `openspec/specs/lab-runner/spec.md` changes.

## Impact

- Affected code: new `internal/progress/`; `internal/runner/session.go`
  (`AttemptIndex` added, `StartedAt` removed); `internal/runner/runner.go`
  (`launch` takes an attempt index, `Start`/`Check`/`Reset`/`Stop` call
  into `internal/progress`); new `breaklab status` subcommand in
  `cmd/breaklab/`.
- Affected data: new `progress/progress.json`, created on first
  `breaklab start` and committed thereafter (not gitignored).
- No new Go dependencies (timestamps and streak math use the standard
  library).
- No impact on `labs/`, the public site, postmortems, or the web UI —
  those are unaffected or come later.
