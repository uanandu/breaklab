# Proposal

## Why

A learner stuck on a lab currently has no path forward except reading the
ticket again or giving up. This is change 4 of 9 in the v1 build plan: it
adds `breaklab hint`, which reveals one hint at a time — a lab's general
hints first, then the active variant's own hints — so a learner can ask
for progressively more specific help without the CLI ever naming the
breakage outright.

## What Changes

- `breaklab hint`: reveals the next unseen hint for the active lab and
  variant. General hints (`lab.yaml`'s top-level `hints:`) are shown
  before the active variant's own hints (`variants[].hints:`), in the
  order each list is written.
- `Session` (`.breaklab/session.json`) gains a `hints_used` count, so the
  CLI knows which hint comes next and survives across separate `breaklab
  hint` invocations (each is its own process).
- Running out of hints: when every hint for the active lab/variant has
  been shown, `breaklab hint` says so and suggests `breaklab reset`,
  rather than erroring or repeating a hint.
- `breaklab reset` clears `hints_used` back to zero along with rebuilding
  the container, since a fresh attempt gets a fresh run at the hints too.
- Out of scope: showing a hint count anywhere outside the session (that's
  progress tracking, change 5), postmortems, and the web UI.

## Capabilities

### New Capabilities

_None._

### Modified Capabilities

- `lab-runner`: adds the **Tiered hints** requirement (reveal order,
  counting, running out). No existing requirement's text changes, but
  **Reset and stop**'s behavior gains the hints-used reset described
  above as an implementation detail of "fresh container," not a new
  scenario — see design.md for why this isn't written as a spec change.

## Impact

- Affected code: new `Hint` method on `internal/runner.Runner`; `Session`
  struct gains `HintsUsed int`; new `breaklab hint` subcommand in
  `cmd/breaklab/`.
- Affected data: none — hints already exist in `web-down`'s `lab.yaml`
  (2 general, 1 per variant) from changes 1–3; no new lab files needed.
- No new Go dependencies, no Docker interaction (hints are read from
  already-loaded catalog data and the session file).
- No impact on the public site, progress tracking, postmortems, or the
  web UI — those are later changes.
