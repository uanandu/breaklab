# Proposal

## Why

`breaklab start` always applies a lab's first variant, so every attempt at
`web-down` breaks the same way. This is change 3 of 9 in the v1 build plan:
it adds weighted random variant selection with a no-immediate-repeat rule,
`--seed` for reproducible picks, and `BREAKLAB_FORCE_VARIANT` for tests —
and fills in `web-down`'s two remaining variants (`port-taken`, `bad-perms`)
and their solution scripts so there is something real to select between.

## What Changes

- New `internal/variants` package implementing weighted random selection:
  pick a variant from a lab's `variants` list weighted by `weight`, never
  repeating the previous attempt's variant when the lab has two or more
  variants, seedable for reproducible picks, and overridable entirely by
  the `BREAKLAB_FORCE_VARIANT` environment variable.
- `breaklab start <lab> [--seed N]`: `--seed` makes selection deterministic
  for that invocation (same seed, same pick, every time).
- A small persisted record of the last variant used per lab, read before
  selection and written after a successful start, so the no-repeat rule
  holds across `breaklab stop` and process restarts, not just within one
  session. Lives next to `.breaklab/session.json`, gitignored, and holds
  nothing sensitive (lab id -> variant id).
- `labs/web-down/variants/port-taken.sh` and
  `labs/web-down/variants/bad-perms.sh`, plus
  `labs/web-down/solutions/port-taken.sh` and
  `labs/web-down/solutions/bad-perms.sh`, matching the "Starter labs" table
  in `docs/breaklab-spec.md` (port 80 taken by another process; homepage
  file permissions). `config-typo` and its solution already exist from
  change 2 and are unchanged.
- `internal/runner.pickVariant` is replaced by a call into
  `internal/variants`; `Runner.Start` and `Runner.Reset` pass through the
  lab's id, its variants, the persisted last-used record, and an optional
  seed.
- Lab integration test extended to cover all three web-down variants (was
  only `config-typo`), using `BREAKLAB_FORCE_VARIANT` to pin each run.
- Out of scope: hints (`breaklab hint`), progress tracking, postmortems,
  the web UI, and any other lab besides `web-down`.

## Capabilities

### New Capabilities

_None._

### Modified Capabilities

- `lab-runner`: adds the **Random variant selection** requirement (weighted
  pick, no-immediate-repeat, `--seed`, `BREAKLAB_FORCE_VARIANT`). The
  existing **Starting a lab** requirement's "Clean start" scenario already
  says "the selected variant script runs inside it" — this change is what
  makes that selection real instead of always picking the first variant,
  but the requirement text itself does not change.

## Impact

- Affected code: new `internal/variants/`; `internal/runner/runner.go`
  (`pickVariant` replaced, `Start`/`Reset` call signatures extended),
  `internal/runner/session.go`-adjacent new history file, new `--seed`
  flag in `cmd/breaklab/lab.go`.
- Affected data: new `labs/web-down/variants/port-taken.sh`,
  `labs/web-down/variants/bad-perms.sh`,
  `labs/web-down/solutions/port-taken.sh`,
  `labs/web-down/solutions/bad-perms.sh`; new gitignored
  `.breaklab/history.json` (or equivalent) created at runtime.
- No new Go dependencies (selection uses `math/rand/v2` from the standard
  library).
- No impact on the public site, the local web UI, progress tracking, or
  postmortems — those are later changes.
