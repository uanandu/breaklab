# Design

## Context

Today `internal/runner.pickVariant` (`internal/runner/runner.go:86`) always
returns `lab.Variants[0]` — a placeholder left by change 2
(`add-lab-runner`) until this change lands. `web-down` has three variants
declared in `lab.yaml` (`config-typo`, `port-taken`, `bad-perms`) but only
`config-typo` has a script and solution on disk; the other two are
metadata-only. `.breaklab/session.json` tracks the *currently active*
attempt and is deleted by `breaklab stop`, so it cannot by itself answer
"what variant did the user see last time?" once a lab is stopped — which
is exactly what the no-immediate-repeat rule needs to know. See
`proposal.md` for the full motivation and `specs/lab-runner/spec.md` for
the exact requirement text this change adds.

## Goals / Non-Goals

**Goals:**
- Pick a variant at random, weighted by `weight`, whenever `breaklab
  start` runs without `BREAKLAB_FORCE_VARIANT` set.
- Never repeat the immediately previous variant for a lab with 2+
  variants, using a record that survives `breaklab stop` and process
  restarts.
- Make selection reproducible with `--seed` and fully overridable with
  `BREAKLAB_FORCE_VARIANT`, for tests and for debugging a specific
  variant by hand.
- Give `web-down` real `port-taken` and `bad-perms` variant + solution
  scripts so there's something to select between.

**Non-Goals:**
- Hints (`breaklab hint`), progress tracking, or postmortems — changes
  4–6.
- Any lab besides `web-down`.
- A general "attempt history" data model. `progress/progress.json`
  (change 5) will eventually record every attempt; this change adds only
  the one small fact (last variant per lab) that selection itself needs,
  so it isn't blocked on progress tracking landing first.

## Decisions

**1. Weighted pick: cumulative-weight walk over `math/rand/v2`.**
Sum the candidate variants' weights, draw one integer in `[0, total)`
with `rand.N`, and walk the list subtracting weights until the draw falls
inside a variant's share. This is the standard approach for weighted
random choice, needs no extra dependency (`math/rand/v2` is stdlib since
Go 1.22), and is easy to unit-test by running it thousands of times and
checking the observed frequencies roughly match the weights. A weight of
`0` is a valid "currently disabled but keep the metadata" marker: it gets
0 share of the draw and is only ever picked if it's the sole remaining
candidate after the no-repeat exclusion (see Decision 3).

**2. Persist "last variant used" in its own file, not inside
`session.json`.**
`.breaklab/history.json` holds a flat `{"<lab-id>": "<variant-id>"}` map,
written only after a *new* selection succeeds (i.e., inside `Start`,
never `Reset`, which intentionally keeps the same variant) and read
before the next selection. Keeping it separate from `session.json` means
`breaklab stop` — which deletes the session — doesn't also erase the
one fact the no-repeat rule depends on. It's gitignored alongside
`.breaklab/session.json` (same directory, same privacy/non-goal: this is
machine-local run state, not progress data meant to be committed).

**3. No-repeat exclusion happens before weighting, and is skipped if it
would leave no candidates.**
Build the candidate list as "all variants except the one named in
history.json for this lab," then weight-pick from that list. Since the
requirement only applies "when the lab has two or more variants," a
2-variant lab always has exactly one candidate left after exclusion
(always correct, never a coin flip), and a 1-variant lab skips exclusion
entirely and always returns that one variant. If every remaining
candidate has weight 0 (all non-previous variants explicitly disabled),
selection falls back to a uniform pick among them rather than erroring —
picking *a* fresh variant is more useful than refusing to start a lab.

**4. `--seed` bypasses the no-repeat exclusion; `BREAKLAB_FORCE_VARIANT`
bypasses both exclusion and weighting.**
The spec's own scenario is "`breaklab start web-down --seed 42` twice
picks the same variant both times" — with no mention of calling `stop` or
otherwise resetting history in between. If exclusion stayed active, the
first call's pick would get written to history and could change what the
*second* call excludes, breaking reproducibility for any lab with 3+
variants. Making `--seed N` mean "seed the PRNG with N and weight-pick
over *all* variants, ignoring history" is the only reading that makes the
scenario true unconditionally, matches how `BREAKLAB_FORCE_VARIANT`
already overrides "the selection rules" in the same requirement, and
still gives `--seed` real value for tests that want "same weights,
reproducible draw" without needing to care about a lab's run history. See
Open Questions for the one wrinkle this creates for the no-repeat rule's
own test coverage.
Seeding: `rand.New(rand.NewPCG(seed, seed))` from the given `--seed`
int64, reused as both PCG seed halves — good enough for test
determinism, not a cryptographic use.

**5. `BREAKLAB_FORCE_VARIANT` must still name a real variant of the lab.**
The env var's value is checked against `lab.Variants` by id (not just the
`^[a-z0-9-]+$` shape check the catalog already applies to every variant
id at load time); an unknown value is a clear startup error rather than
a silent fallback to random selection, so a typo in a test script fails
loudly instead of passing against the wrong breakage.

**6. `port-taken` and `bad-perms` follow the exact shape of
`config-typo`.**
`docs/breaklab-spec.md`'s "Starter labs" table already names these two
("port 80 taken by another process", "homepage file permissions") and
explicitly says web-down is the one lab where reading its own spec
doc's variant list is fine, unlike every later lab. `port-taken.sh`
starts a second process bound to port 80 before nginx can claim it;
`bad-perms.sh` revokes read access on the homepage file from the user
nginx's worker runs as. Each gets a matching `solutions/*.sh` that undoes
exactly that change (free the port / restore permissions), used only by
the lab integration test, never by the CLI.

## Risks / Trade-offs

- **[Risk]** A learner could read `.breaklab/history.json` by hand and
  infer that *some* other variant will be picked next, narrowing their
  guess before starting the lab. → **Mitigation**: it only ever names the
  *previous* variant, never the next one, and the file lives next to the
  already-gitignored `session.json` under the same "local run state,
  not a secret, not committed" treatment the spec already accepts for
  session state. No stronger hiding than that is in scope here.
- **[Risk]** `math/rand/v2`'s global functions are safe for concurrent
  use, but a seeded `rand.New(...)` instance created per call is not
  shared — fine for this CLI (one command per process), but would need
  revisiting if `breaklab serve`'s future web UI ever triggered starts
  concurrently. → **Mitigation**: out of scope; note left here for
  change 8 (`add-local-web-ui`) to re-check.
- **[Trade-off]** Falling back to a uniform pick when every non-excluded
  candidate has weight 0 is a judgment call the spec doesn't make
  explicit (see Open Questions). → **Mitigation**: documented here and
  in code comments at the point of the fallback, so it's easy to revisit
  if a future lab actually uses weight 0 deliberately and this guess
  turns out wrong.

## Spec clarifications worth flagging

Per the request to call out anything in the spec that reads unclear:

1. **The `--seed` scenario doesn't say whether it interacts with the
   no-repeat rule.** Decision 4 above resolves it by having `--seed`
   bypass history-based exclusion entirely. An alternative reading —
   exclusion stays active, and the scenario only holds for 2-variant labs
   — would also satisfy the literal text (web-down's test could use a
   2-variant fixture) but makes `--seed` silently behave differently
   depending on how many variants a lab happens to have, which seems more
   surprising than documenting the bypass. Flagging this in case the
   intent was actually the narrower reading.
2. **Weight `0` isn't addressed by the spec at all.** `lab.yaml`'s schema
   allows any integer `weight`, and nothing in the requirements says what
   `0` (or a negative number) should do. Decision 1 and 3 above treat `0`
   as "never preferred, only picked if forced by exclusion" and don't
   address negative weights beyond treating them as invalid input
   (rejected with an error at selection time, same as an unknown
   `BREAKLAB_FORCE_VARIANT`).
3. **"the same variant used on that lab's previous attempt" — previous
   *attempt*, not previous *start*.** `breaklab reset` keeps the same
   variant and doesn't count as a new attempt for this purpose (Decision
   2: `Reset` never writes `history.json`). If "previous attempt" was
   meant to include resets too, the behavior is the same either way,
   since the variant doesn't change on reset — but the distinction
   matters once progress tracking (change 5) defines "attempt" formally,
   so it's worth confirming they end up consistent.

## Open Questions

- Should a future `breaklab status`-style view (change 5+) ever surface
  "last variant was X" to the user directly, or should that stay
  internal to selection the way the active variant id itself stays
  hidden until solved? Doesn't affect this change's specs or tasks either
  way — can be decided when progress tracking is designed.
