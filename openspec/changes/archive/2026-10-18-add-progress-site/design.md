# Design

## Context

Every file this change reads already exists and is already committed:
`roadmap.yaml`, `labs/*/lab.yaml` (via the catalog), `progress/progress.json`
(change 5), and `postmortems/*.md` (change 6). Nothing here needs Docker,
a session, or any of `internal/runner`'s machinery — per
`docs/breaklab-spec.md`'s own architecture, `site/site.go` is listed as
its own concern alongside `runner/runner.go`, not inside it. See
`proposal.md` for the full motivation and `specs/progress-site/spec.md`
for the exact requirement text.

## Goals / Non-Goals

**Goals:**
- `breaklab publish`: a from-scratch rebuild of `site/` from committed
  data only.
- A GitHub Actions workflow that runs it on every push to `main` and
  deploys the result to GitHub Pages.

**Non-Goals:**
- The local web UI (change 8) — a different reader of the same files,
  built later, served live instead of as a static build.
- Pinning third-party Actions to commit SHAs — the build plan assigns
  that to change 9's security audit, which explicitly lists
  `publish.yml` as something it will harden; doing it here would be
  redundant work change 9 repeats, not a security gap this change
  leaves open given the plan's own ordering.
- Real Markdown rendering (bold, links, code spans, lists) for
  postmortem bodies — see Decision 6.

## Decisions

**1. A "complete" lab's date, difficulty, and minutes-to-fix always show
on the site; only the postmortem *link* (and its content) is gated by
`publish: true`.**
The "Nothing leaks" scenario's own wording is specific: "no page, link,
or **text from it**" — "it" being the postmortem. A lab's date,
difficulty (from `lab.yaml`, already public information about the lab
itself), and minutes-to-fix (derived from `progress.json`'s own
timestamps, not from the postmortem file) aren't "from" the postmortem
at all — they're facts about the attempt, already visible to the
learner locally via `breaklab status`. Hiding them would make an
unpublished lab vanish from its stage's list entirely, which the spec's
"lists completed labs" bullet doesn't say is conditional. The
alternative reading — unpublished means the whole row disappears —
is flagged below since it's an equally defensible, more conservative
choice.

**2. "Labs completed" (on the public site) counts only attempts with
status `complete`, not `solved`.**
`breaklab status` (change 5) counts `solved` *or* `complete` as done,
because when it was built postmortems didn't exist yet and "solved"
was the only finish line available. The public site exists specifically
to show finished, written-up work — showing a lab as "done" on a page
meant to be shared, with no postmortem ever written for it, undersells
what "complete" is supposed to mean here. This is a new,
`progress-site`-local completion count (not a change to
`progress.StageCompletion`, which keeps its existing `breaklab status`
behavior unchanged — see Impact).

**3. "Variants seen" counts distinct (lab, variant) pairs across every
attempt, any status.**
The spec doesn't define it. Counting only solved/complete attempts would
make this number identical to a derivative of "labs completed" in most
cases (not meaningfully different information); counting every attempt
regardless of outcome — including ones later abandoned — answers a
different, more interesting question: "how many different breakages has
this learner actually been thrown at," which is exactly what varies
*because* of change 3's random selection and is otherwise invisible on
this page.

**4. Postmortem bodies become HTML by reusing
`internal/postmortem`'s existing section splitting — heading plus
blank-line-separated paragraphs — not a Markdown parser.**
The project has added zero new Go dependencies through six changes so
far, and nothing in the spec says postmortems need inline Markdown
(bold, links, code spans) rendered correctly — the five required
sections are prose paragraphs, consistent with
`templates/postmortem.md`'s own plain-paragraph placeholder text. A
postmortem's rendered page shows each of `postmortem.RequiredSections`'
section as an `<h2>` followed by one `<p>` per blank-line-separated
block of its (already-validated, 20+-word) text, through `html/template`'s
normal auto-escaping. A learner who wrote `**bold**` in their own
postmortem sees the literal asterisks on the public page, not bold text —
accepted as a known, minor cosmetic gap rather than a reason to add a
Markdown library.

**5. `breaklab publish` wipes `site/` (`os.RemoveAll` then rebuild) on
every run, rather than only adding or updating files.**
Without this, a postmortem published once and later flipped back to
`publish: false` would leave its old page sitting in `site/postmortems/`
forever, directly violating "no page, link, or text from it appears
anywhere in site/" the next time someone happens to find the stale URL.
A full rebuild costs nothing extra here (the whole site is small,
generated in milliseconds) and makes "nothing leaks" true by
construction instead of by remembering to clean up.

**6. `.github/workflows/publish.yml` uses version tags
(`actions/checkout@vN`, etc.), not pinned commit SHAs.**
See Non-Goals: the build plan's change 9 task list already names
`publish.yml` as something it pins. Pinning here would be work this
change does and change 9 immediately audits/potentially redoes anyway;
leaving tags for now keeps this change's scope matched to "ship
`breaklab publish` and get it deploying," with the hardening pass
exactly where the plan put it.

**7. The workflow builds the Go binary and runs it directly
(`go build ./cmd/breaklab && ./breaklab publish`), no Docker step.**
`breaklab publish` never touches Docker (Decision in Context), so the
workflow only needs a Go toolchain — faster and simpler than every other
command's requirements, and consistent with `breaklab publish` not
accepting `--help`-documented Docker-related flags or errors.

## Risks / Trade-offs

- **[Trade-off]** Decision 1 means a learner who solves a lab but never
  publishes its postmortem still shows that lab's difficulty and time-to-
  fix publicly (just not the write-up). → **Mitigation**: accepted;
  `publish: false` is explicitly about the postmortem's own content per
  the spec's wording, and difficulty/time-to-fix are coarse enough not
  to be sensitive — nothing a resume-style page wouldn't already imply
  by listing the lab as done.
- **[Risk]** Decision 2 means `breaklab status`'s "labs completed" and
  the public site's "labs completed" can disagree (a learner with three
  `solved`-but-not-written-up labs sees 3 fewer on the site than
  locally). → **Mitigation**: that's the intended signal — the site
  is explicitly "truly finished, written up" — but it's worth the
  learner knowing the two numbers measure different things, which this
  design note exists to make explicit rather than let be a silent
  surprise.

## Spec clarifications worth flagging

Per the request to call out anything in the spec that reads unclear:

1. **Whether an unpublished postmortem hides its whole completed-lab
   row, or just its own link/content, isn't stated.** Decision 1 picks
   the less conservative reading; the alternative (hide the entire row)
   is one `if` away if that turns out to be the intent.
2. **"variants seen" isn't defined at all.** Decision 3 resolves it;
   flagging because "distinct variants among completed labs only" is an
   equally plausible, narrower reading that would make the number
   smaller and more directly tied to "labs completed."
3. **Whether "labs completed" means `solved`-or-`complete` (matching
   `breaklab status`) or `complete`-only wasn't settled by change 5**,
   since postmortems didn't exist when that spec was written. Decision 2
   resolves it for the public site specifically, while leaving
   `breaklab status`'s own, already-shipped behavior untouched.
4. **The spec never mentions how postmortem Markdown becomes HTML**, or
   whether a Markdown dependency is expected. Decision 4 resolves it
   with the plainest possible approach that still satisfies every
   stated scenario.
5. **Nothing in the spec says old pages must be cleaned up on
   republish.** Decision 5 resolves it; the alternative (an additive
   build that never deletes) would technically satisfy every requirement
   on a system that's never had a postmortem's `publish` flag flipped
   back to `false` after being `true` — which is exactly the case
   "nothing leaks" seems written to guard against.

## Open Questions

None — every ambiguity found had a specific, documented resolution
above.
