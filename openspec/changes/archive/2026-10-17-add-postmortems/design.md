# Design

## Context

Since change 5, `progress.json`'s `Attempt` struct already declares
`Postmortem string` and a `"complete"` status, but nothing writes either
— every solved attempt stays `"solved"` forever. This change is what
finally uses both fields: it picks an eligible attempt, writes a file for
it, and on a passing validation promotes it to `"complete"`. See
`proposal.md` for the full motivation and `specs/postmortems/spec.md` for
the exact requirement text.

## Goals / Non-Goals

**Goals:**
- `breaklab postmortem`: create a postmortem for the most recently
  solved attempt that doesn't have one yet, from `templates/postmortem.md`,
  and open it in `$EDITOR`.
- `breaklab postmortem --validate [path]`: check the five required
  sections have real content, and promote the attempt to `"complete"`
  on success.

**Non-Goals:**
- Reading `publish: true` to actually build anything — that's `breaklab
  publish` and the public site (change 7).
- Any web UI postmortem viewer (change 8).
- Re-opening or re-creating a postmortem that already exists for an
  attempt — once `Attempt.Postmortem` is set, that attempt is simply no
  longer "eligible" for a *new* one; editing an existing file is just
  opening it directly, which doesn't need this CLI's help.

## Decisions

**1. "Most recently solved attempt" means: the attempt with the latest
`solved_at` among those with status `solved` or `complete` and an empty
`Postmortem` field.**
Requiring `Postmortem == ""` is what makes "Nothing solved... since the
last postmortem" true without needing a separate "last postmortem"
pointer: as soon as `breaklab postmortem` creates a file for an attempt,
it writes that path onto the attempt record (Decision 2), which removes
it from future selection. Including `"complete"` attempts in the
candidate pool alongside `"solved"` ones is defensive, not reachable
today (a `"complete"` attempt already has a non-empty `Postmortem` by the
time it got promoted — Decision 6), but it costs nothing and avoids a
subtle bug if that invariant ever changes.

**2. `Attempt.Postmortem` is set the moment the file is created, before
the learner has written anything.**
This is what makes the attempt ineligible for Decision 1's selection on
a second `breaklab postmortem` call, and it's also the one piece of
state `--validate`'s no-path case (Decision 4) needs to find "the
postmortem I just made." The trade-off: if the learner deletes the file
without ever validating it, the attempt is now permanently ineligible
for `breaklab postmortem` to recreate (see Risks).

**3. The filename's date is the attempt's `solved_at` date (learner's
local calendar day), not today's date.**
`docs/breaklab-spec.md`'s own template example shows `date: 2026-10-17`
alongside a doc dated "Oct 17, 2026," suggesting the date is "when this
happened," not "when I got around to writing it up." Using `solved_at`
also means the date in the filename and the date in the front matter are
always the same value, computed once — whereas "today" could disagree
with `solved_at` if the learner solves a lab one day and runs `breaklab
postmortem` the next.

**4. `breaklab postmortem --validate` with no path validates the most
recently *created* postmortem — the latest attempt (by `solved_at`) with
a non-empty `Postmortem`, regardless of whether it's already `complete`.**
The spec's own scenario phrasing — "runs `breaklab postmortem --validate`
on a file" — reads as if a path is normally expected, but doesn't specify
what happens without one, and the natural use right after writing one
is "check what I just wrote, without having to retype the path." Not
excluding already-`"complete"` attempts means re-running `--validate`
after further edits keeps checking the same file rather than silently
drifting to an older one.

**5. `--validate <path>` (path given) validates exactly that file and,
on success, promotes whichever attempt's `Postmortem` field matches that
path — not necessarily the most recent one.**
This is what makes `--validate` useful for more than just the file you
just created (re-checking an older draft, or running it from a script),
and it's a strict superset of Decision 4's behavior once a path happens
to match the latest attempt.

**6. Promoting `"solved"` to `"complete"` reuses change 5's idempotent,
status-gated pattern.**
New `progress.MarkComplete(path string, index int) error` only overwrites
a record whose status is currently `"solved"` (mirrors
`MarkSolved`/`MarkAbandoned`'s "only while active" guard from change 5,
generalized to "only while still in the state this transition expects").
Running `--validate` again on an already-`"complete"` postmortem is a
safe no-op, not an error.

**7. `$EDITOR` is split into argv with `strings.Fields`, never passed to
a shell.**
Matching the project's "never shell out through `sh -c`" rule:
`strings.Fields(os.Getenv("EDITOR"))` splits `"code --wait"` into
`["code", "--wait"]`, the postmortem's path is appended as the final
argument, and the result is run via `exec.CommandContext` with stdio
inherited so the editor is interactive. An empty `$EDITOR` falls back to
`["nano"]`. This doesn't handle an editor command whose own arguments
contain quoted spaces (e.g. a path with a space baked into an alias) —
accepted, since that's a shell-quoting problem this project's "no shell"
rule explicitly refuses to solve, and `$EDITOR` is conventionally a
bare command plus simple flags.

**8. A filename collision (same lab, variant, and solved-at date) gets a
`-2`, `-3`, ... suffix rather than being overwritten or rejected.**
Starting the same lab, cycling back to the same variant, and solving it
again on the same calendar day is unlikely but possible once there are 3+
variants (change 3's no-immediate-repeat rule only forbids repeating the
*previous* variant, not cycling back after others). Silently overwriting
an earlier postmortem would destroy the learner's own writing; refusing
to create a new one would make `breaklab postmortem` error for a
perfectly legitimate second solve. A numeric suffix keeps both files and
needs no new state beyond checking whether the base path already exists.

**9. `Runner.Postmortem`/`Runner.ValidatePostmortem` need no Docker,
like `Hint` and `Status`.**
Both only touch the catalog (for `stage`), `progress.json`, and the
filesystem. `$EDITOR`'s invocation is injected as a `Runner.OpenEditor
func(ctx, path string) error` field (defaulting to the real
`exec.CommandContext`-based implementation), the same pattern as
`HostExecFunc`, so tests don't have to actually launch an editor.

## Risks / Trade-offs

- **[Risk]** Decision 2 means deleting a just-created, still-blank
  postmortem file leaves its attempt permanently ineligible for
  `breaklab postmortem` to offer again (its `Postmortem` field is set,
  even though the file is gone). → **Mitigation**: accepted as a sharp
  edge for v1; the learner can always create the file by hand at the
  recorded path, or clear `Attempt.Postmortem` directly in
  `progress.json` (a committed, human-editable file by design). A
  friendlier recovery path (e.g. detecting a missing file and treating
  it as eligible again) is a reasonable follow-up, not required now.
- **[Trade-off]** Decision 3 ties the filename to `solved_at`'s local
  date, so a postmortem written weeks after solving still files under
  the original date, which could look odd in a plain directory listing
  sorted by name vs. by file mtime. → **Mitigation**: accepted; the
  front matter's `date` field matches the filename either way, so
  there's no inconsistency in the file's own content, only in how old it
  looks compared to when it was actually typed.

## Spec clarifications worth flagging

Per the request to call out anything in the spec that reads unclear:

1. **The "Publishing control" requirement in `docs/breaklab-spec.md` has
   no scenario at all** — the markdown block ends right after the
   requirement's description. `openspec validate` requires at least one
   scenario per requirement, so `specs/postmortems/spec.md` adds one
   scoped to what *this* change actually builds (new postmortems default
   to `publish: false` and nothing in this change ever flips it) rather
   than inventing a "and therefore the site only shows published ones"
   scenario that belongs to change 7's `progress-site` spec, which
   already has its own "Nothing leaks" scenario covering that
   enforcement.
2. **"most recently solved attempt" doesn't define what happens once
   that attempt already has a postmortem.** Decision 1 resolves it by
   folding "doesn't have one yet" into the definition of "eligible,"
   which is also what makes the "Nothing solved... since the last
   postmortem" wording in the spec's own second scenario literally true.
3. **`--validate`'s no-path behavior is unstated.** Decision 4 resolves
   it; flagging because an alternative reading — `--validate` always
   requires a path — would make the CLI signature simpler but less
   convenient for the obvious "check what I just wrote" use.
4. **How `minutes_to_fix` is computed isn't restated here** since
   change 5 already settled it: it's `solved_at - started_at` on the
   attempt record, which (per change 5's Decision 2) spans every
   `breaklab reset` in between — this change just formats that duration
   as whole minutes for the front matter, with no new computation to
   design.
5. **Filename collisions aren't mentioned in the spec at all.**
   Decision 8 resolves it with a numeric suffix; flagging since an
   equally defensible choice (error and tell the learner to rename
   manually) would also satisfy "don't silently destroy prior work" but
   is less convenient.

## Open Questions

None — every ambiguity found had a specific, documented resolution above.
