# Design

## Context

`labs/web-down/lab.yaml` already declares two general hints and one hint
per variant (from change 1); nothing reads them yet. `.breaklab/session.json`
(`internal/runner/session.go`) already tracks the active lab, container,
and variant across separate CLI invocations — each `breaklab` command is
its own process, so anything `breaklab hint` needs to remember between
calls (how many hints it has already shown) has to live there too, the
same way `internal/runner/history.go` (change 3) added a file next to it
for a different cross-invocation fact. See `proposal.md` for the full
motivation and `specs/lab-runner/spec.md` for the exact requirement text.

## Goals / Non-Goals

**Goals:**
- Reveal exactly one new hint per `breaklab hint` call, in a fixed order:
  all of the lab's general hints, then all of the active variant's hints.
- Remember how many hints have been shown for the active attempt, so
  repeated calls progress instead of repeating.
- Tell the learner plainly when hints have run out and point them at
  `breaklab reset`.

**Non-Goals:**
- Surfacing a hints-used count anywhere outside `breaklab hint` itself
  (no `breaklab status`, no web UI) — that's progress tracking (change 5)
  and the web UI (change 8).
- Any new lab content. `web-down` already has hints from change 1.
- Docker interaction of any kind — hints are pure data (catalog + session
  file), unlike every other lab command so far.

## Decisions

**1. `Hint` needs no Docker, unlike every other lab command.**
`start`/`check`/`reset`/`stop` all call `r.Docker.Ping` because they act
on a container. `breaklab hint` only reads the active lab's `lab.yaml`
(already loaded by the catalog) and the session file, so `Runner.Hint`
skips the Docker dependency entirely — it works even if the Docker
daemon happens to be down, which is a small but real convenience for a
learner who's mid-lab and just wants a nudge.

**2. Hint order is "all general hints, then all variant hints," each in
the order written in `lab.yaml`.**
The requirement's own wording — "general hints first, then the active
variant's hints" — reads as two already-ordered lists concatenated, not
an interleaving or a re-sort by any other key. `web-down`'s two general
hints ("what does curl return", "is the process running") are
deliberately broader than any variant's one hint ("ask nginx to test its
own config"), so concatenation naturally goes coarse-to-fine without the
CLI needing to know anything about hint content.

**3. `hints_used` lives on `Session`, not in a new file.**
Unlike change 3's "last variant used" (which has to outlive `breaklab
stop` so the *next* attempt can see it), how many hints have been shown
is only meaningful *during* the current attempt — `breaklab stop` ending
the attempt and `breaklab reset` starting a fresh one both make the old
count meaningless. That's exactly `Session`'s existing lifetime, so
`HintsUsed int` is a new field on the existing struct rather than another
file. `ClearSession` (unchanged) already drops it on `stop`.

**4. `breaklab reset` zeroes `hints_used`.**
The existing **Reset and stop** requirement calls `reset` "same variant,
fresh container" — a fresh attempt. A learner resetting is explicitly
asking to try again, and having `breaklab hint` immediately say "out of
hints, try `breaklab reset`" right after they just did that would be a
dead end. Since `reset` already writes a new session (same code path as
`start`, via `launch`), zeroing `HintsUsed` there is a one-line change to
`Reset`'s existing session write, not a new scenario — see "Spec
clarifications" below for why this isn't written as a MODIFIED
requirement.

**5. `breaklab hint` with no active lab errors like `breaklab check`
does.**
The spec doesn't say what happens with no lab running, but a hint is
meaningless without a lab and variant to hint about. Reusing the
existing `ErrNoActiveLab` (already used by `Check`) gives the same
message and exit code learners already see from `check`, rather than
inventing a second "nothing is running" error.

**6. A variant with stale/missing data after a `lab.yaml` edit errors
rather than silently showing fewer hints.**
`Reset` already has this guard ("variant of the active session no longer
exists in lab %q") for the case where someone edits `lab.yaml` mid-attempt
and the active variant disappears. `Hint` reuses the same check before
reading that variant's hints, for the same reason: silently treating a
missing variant as "no variant hints" would hide a real problem (a
broken lab.yaml) behind what looks like a lab with fewer hints than it
has.

## Risks / Trade-offs

- **[Risk]** If a future lab has zero general hints and zero variant
  hints, the very first `breaklab hint` call immediately says "out of
  hints." → **Mitigation**: that's the correct behavior per the spec (no
  hints exist, so none are shown), not a bug; worth a lint check in a
  later change (`add-security-audit` or a catalog validation change)
  flagging labs with no hints at all, but out of scope here.
- **[Trade-off]** Resetting `hints_used` on `breaklab reset` (Decision 4)
  means a learner can "farm" hints by resetting repeatedly right after
  exhausting them, rebuilding the container each time just to re-read the
  same hints. → **Mitigation**: accepted; this is a personal training
  tool on the honor system (see `docs/breaklab-spec.md`'s "Non-goals: no
  anti-cheating"), and the alternative (hints_used surviving reset)
  directly contradicts "fresh container" meaning a fresh attempt.

## Spec clarifications worth flagging

Per the request to call out anything in the spec that reads unclear:

1. **The spec never says what `breaklab reset` does to the hint count.**
   Decision 4 resolves this by zeroing it, but the literal requirement
   text only covers `breaklab hint`'s own behavior. Flagging in case the
   intent was for hints-used to persist across a reset (e.g. to cap total
   hints per lab regardless of retries) — that would be a materially
   different design (hints_used would need to move to the change-3-style
   persisted history file instead of the session), so worth confirming
   before change 5 (progress tracking) locks in "attempt" semantics that
   assume one or the other.
2. **"the active variant's hints" presumes a variant is always active
   when `breaklab hint` runs.** Decision 5 fills the gap (no active lab
   errors like `check` does) since the spec only describes the
   steady-state case.
3. **No scenario for the general-hints-only case (a lab with one
   variant and no variant-specific hints, or vice versa).** Decision 2's
   plain concatenation handles this for free — an empty list contributes
   nothing to the sequence — so no special-casing was needed, but it's
   worth noting the spec's single scenario doesn't exercise it directly.

## Open Questions

None — Decision 4's reset behavior is the one judgment call with real
design weight, and it's resolved above rather than deferred, since it
determines what field `HintsUsed` is on and how `Reset` is implemented.
