# Proposal

## Why

Nothing today makes a learner explain *why* a lab broke and how they
found it — the part that actually sticks, and the part worth reusing in
an interview. This is change 6 of 9 in the v1 build plan: it adds
`breaklab postmortem`, which turns the most recently solved, not-yet-
written-up attempt into a filled-in template, validates that the
required sections are really written (not left as placeholders), and
only then promotes that attempt from `solved` to `complete` in
`progress.json`.

## What Changes

- New `templates/postmortem.md`: the front-matter-plus-sections template
  shown in `docs/breaklab-spec.md`, copied for each new postmortem.
- New `internal/postmortem` package: picks the attempt a postmortem is
  for, renders the template, opens it in `$EDITOR` (falling back to
  `nano`), and validates a postmortem's five required sections each have
  20+ words of real content once the placeholder HTML comments are
  stripped out.
- `breaklab postmortem`: creates `postmortems/<date>-<lab-id>-<variant-id>.md`
  for the most recently solved attempt that doesn't have one yet, with
  front matter (`lab`, `variant`, `stage`, `date`, `minutes_to_fix`,
  `hints_used`, `publish: false`), and opens it in `$EDITOR`.
- `breaklab postmortem --validate [path]`: validates the five required
  sections of the given file, or — with no path — the most recently
  created postmortem. On success, promotes that attempt's status from
  `solved` to `complete` in `progress.json` (new `progress.MarkComplete`,
  following the same idempotent, status-gated pattern as change 5's
  `MarkSolved`/`MarkAbandoned`).
- `Attempt.Postmortem` (declared but always empty since change 5) gets
  its first real writer: set to the created file's path as soon as
  `breaklab postmortem` creates it, which is also what makes that
  attempt ineligible to be picked again by a later `breaklab postmortem`
  call.
- "Nothing solved" case: when every solved attempt already has a
  postmortem path recorded, `breaklab postmortem` exits non-zero with
  "Solve a lab first".
- Out of scope: the public progress site reading `publish: true`
  postmortems (change 7) and the web UI's postmortem viewer (change 8).

## Capabilities

### New Capabilities

- `postmortems`: creating a postmortem from the template for the most
  recently solved, not-yet-written-up attempt; validating its required
  sections; and promoting the attempt to `complete` on a valid postmortem.

### Modified Capabilities

_None._ `progress`'s existing requirements (attempt log, status summary)
keep their documented behavior — this change is what finally makes the
`"complete"` status and the `postmortem` field (both declared in change 5
for forward compatibility) get used for the first time, but it adds a
new write path rather than changing what either existing requirement
says, so no requirement text in `openspec/specs/progress/spec.md` changes.

## Impact

- Affected code: new `internal/postmortem/`; `internal/progress/`
  gains `MarkComplete`; `internal/runner/` gains `Postmortem` and
  `ValidatePostmortem` methods (no Docker needed, matching `Hint`/
  `Status`) plus an injectable "open in editor" function for testability;
  new `breaklab postmortem` subcommand in `cmd/breaklab/`.
- Affected data: new `templates/postmortem.md`; new `postmortems/`
  directory, created on first use and committed (not gitignored, per
  `docs/breaklab-spec.md`'s "User data: ... postmortems/*.md (committed)").
- No new Go dependencies (template rendering and `$EDITOR` invocation use
  the standard library).
- No impact on `labs/`, the public site, or the web UI — those are
  unaffected or come later.
