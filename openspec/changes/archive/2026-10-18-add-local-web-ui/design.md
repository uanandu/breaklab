# Design

## Context

Everything this change reads already exists: `roadmap.yaml`,
`labs/*/lab.yaml` (via the catalog), `.breaklab/session.json` (via
`runner.LoadSession`, a free function with no Docker dependency),
`progress/progress.json`, and `postmortems/*.md`. Per
`docs/breaklab-spec.md`'s architecture, `web/web.go` is its own concern,
parallel to `site/site.go`, `runner/runner.go`, etc. — not a Runner
method. `docs/deploy-raspberry-pi.md` already exists in the repo and
already documents `--host`/`--port`, the default port 8787, the
port-in-use error, and the one-time non-loopback warning; this change's
job is to make the implementation match what that doc already promises,
not to write new doc content. See `proposal.md` for the full motivation
and `specs/web-ui/spec.md` for the exact requirement text.

## Goals / Non-Goals

**Goals:**
- `breaklab serve`: a live local server with five views, all read
  straight from committed/session files, no publish step.
- Make `catalog`'s lab status accurate (available/in-progress/complete),
  finally completing what change 1 explicitly deferred.

**Non-Goals:**
- Change 9's formal security audit — this change builds safely by
  construction (validated ids, no path built from request input,
  specific routes over generic file serving), but the audit pass itself,
  CI's `govulncheck`, and action-SHA pinning stay change 9's job.
- Writing or editing `docs/deploy-raspberry-pi.md` — it already exists
  and already documents this change's behavior; only touch it if
  implementation reveals it's wrong about something.
- Any new public-site behavior — `breaklab publish` (change 7) is
  untouched.

## Decisions

**1. No HTMX; the challenge browser's filters are plain HTML forms with
full-page GET requests.**
The project's stack notes name HTMX for the web UI's interactivity, but
after discussion, building that properly would mean vendoring
`htmx.min.js` as a local static asset — a third-party JS file I cannot
fetch or verify against the real release from inside this environment,
written from memory into a file that would then run in the learner's
browser. A `<form method="get">` with `stage`/`difficulty`/`status`
fields, submitted via ordinary browser navigation, satisfies "filter by
stage, difficulty, and status" and "clearing the filters restores the
full list" exactly — full-page reloads instead of partial-page swaps,
with zero JavaScript and zero new files to trust. This is a deliberate,
flagged deviation from the stack notes, not an oversight.

**2. A lab's status is computed from `progress.json`, not stored:
`complete` if it has any `status == "complete"` attempt; `in-progress`
if it has any `"active"` or `"solved"` attempt and no `"complete"` one;
`available` otherwise — including a lab whose only attempts are all
`"abandoned"`.**
`internal/catalog`'s `computeStatus` has been a stub returning
`"available"` unconditionally since change 1, with a comment saying so
explicitly pending progress tracking. Progress tracking has existed
since change 5; this change is what finally reads it. A lab tried and
abandoned, with nothing else, goes back to `available` rather than
sitting in `in-progress` forever — nothing about it is actually still
open, and leaving it "in progress" would make an abandoned attempt look
like unfinished work rather than what it is (a clean slate to retry).

This computation lives in `internal/progress`, not `internal/catalog`,
and is applied as a separate pass over an already-loaded `*catalog.Catalog`
(`progress.ApplyStatus(cat, pf)`, mutating each `Lab.Status` in place) —
**not** inside `catalog.LoadCatalog` itself. `internal/progress` already
imports `internal/catalog` (for the existing `StageCompletion`, which
already combines both types); having `internal/catalog` import
`internal/progress` back, so `LoadCatalog` could compute status directly,
would create an import cycle. `catalog.LoadCatalog`'s own stub keeps
returning `"available"` unconditionally — it has no progress data to do
otherwise with — and every caller that wants accurate status (the web
UI, and `breaklab list`, updated here to load `progress.json` too) calls
`progress.ApplyStatus` right after loading the catalog. This still
completes change 1's deferred stub and still fixes `breaklab list`'s
always-`"available"` display as a direct, intended consequence — just
with the real computation sitting in the package that was already
allowed to depend on both types, rather than the one that wasn't.

**3. `internal/site`'s private "latest complete attempt for a lab"
lookup moves to `internal/progress` as an exported helper; `internal/web`
uses the same one; `internal/web` never imports `internal/site`.**
Both the public site (which lab/attempt counts as "this lab's
completion") and the local dashboard/roadmap need the identical
computation. `internal/progress` is the correct shared home — both
`site` and `web` already depend on it, and neither should depend on the
other (they're sibling concerns per the architecture doc: one builds a
static publish-triggered artifact, the other serves live requests).
A one-line helper `internal/web` needs that `internal/site` happens to
also have (deriving a postmortem's URL slug from its file's basename) is
not worth a cross-package dependency for one line — it gets its own
trivial copy in `internal/web` instead.

**4. The lab detail view never reveals a variant id or any hint text —
"hints used so far" is rendered as a count, never the hints themselves —
regardless of whether the lab is solved; the postmortem viewer is a
separate, more permissive view.**
The CLI already hides the variant id until a lab is solved (change 2);
this view goes further and never shows it at all, matching the spec's
explicit "whether or not the lab is solved" wording. This isn't a
contradiction with the postmortem viewer: once a learner has written and
is viewing their *own* postmortem for a lab, it already contains
`variant: ...` in front matter they wrote themselves, describing the fix
in detail — hiding that there would make the page they wrote worse, not
safer. The variant-hiding rule is scoped to the lab detail view, which
is about a lab the learner may not have solved yet.

**5. Hints-used count and elapsed time for a lab: read live from the
session when it's the currently active lab, otherwise from the latest
attempt's persisted values in `progress.json` (zero if the lab has never
been attempted).**
Same "live while active, snapshotted once finalized" split change 5
already uses for `breaklab status` — reusing a pattern rather than
inventing a second one for the same underlying fact.

**6. The dashboard/roadmap's "labs completed" total counts `complete`
attempts only, matching the public site (change 7) — not
`breaklab status`'s older `solved`-or-`complete` count, which this
change does not touch.**
`breaklab status` predates postmortems and was never updated to
distinguish `solved` from `complete`; the two newer, postmortem-aware
surfaces (the public site and now the local web UI) agree with each
other instead, by design, even though that means the CLI's `status` and
the web UI's dashboard can show different numbers for the same data —
documented here so that's a known, intentional divergence, not a
surprise.

**7. Postmortem viewer routes resolve a requested slug against a map
built only from known complete attempts' actual `Postmortem` paths —
never by concatenating the request path into a filesystem path.**
This is the same shape of defense `breaklab publish`'s own postmortem
loading already uses (change 7): the set of valid postmortems is fully
known in advance from `progress.json`, so a request for anything outside
that known set is a 404, not a file-read attempt — no path-traversal
surface exists to audit later, by construction.

**8. `breaklab serve`'s server lifecycle: `net.Listen` first (for a
clear port-in-use error), then `http.Serve` on that listener, with
graceful shutdown on the `signal.NotifyContext` pattern `commandContext()`
already provides every other command (extended here to catch SIGTERM as
well as SIGINT), and an
OS-specific default-browser opener (`xdg-open`/`open`/`start` by
`runtime.GOOS`) run via `exec.Command` directly — never a shell — only
when the bind host is loopback.**
Listening before serving is what turns a generic "address already in
use" `net.OpError` into the specific, already-promised "port N is
already in use" message `docs/deploy-raspberry-pi.md`'s troubleshooting
section describes. Reusing `commandContext()` means Ctrl-C behaves the
same way here as it already does for every other long(er)-running
command. SIGTERM is added to it because `docs/deploy-raspberry-pi.md`
runs `breaklab serve` as a systemd service, and `systemctl stop` sends
SIGTERM, not SIGINT — without it the service would be killed outright
instead of shutting down cleanly. Every other command benefits the same
way (e.g. a `kill` during `breaklab start` cancels its context rather
than leaving a half-started container behind uncleaned).

## Risks / Trade-offs

- **[Trade-off]** Decision 1's full-page-reload filtering is less fluid
  than HTMX's partial swaps would be. → **Mitigation**: accepted; see
  Decision 1 — the trust cost of vendoring unverifiable third-party JS
  outweighs the UX gain for a filter form.
- **[Risk]** Decision 2 changes `breaklab list`'s displayed output for
  any lab with attempt history (previously always `"available"`, now
  accurate). → **Mitigation**: this is a bug fix completing change 1's
  own documented stub, not a behavior change anyone depended on being
  wrong; flagged here and in the proposal so it isn't missed.
- **[Risk]** Decision 6 means the CLI (`breaklab status`) and the web UI
  can disagree on "labs completed" for a learner with `solved`-but-not-
  written-up labs. → **Mitigation**: accepted, same reasoning as change
  7's identical trade-off for the public site vs. `breaklab status`.

## Spec clarifications worth flagging

Per the request to call out anything in the spec that reads unclear:

1. **Whether "in-progress" should include a lab whose only attempts are
   all abandoned isn't stated.** Decision 2 resolves it as `available`,
   not `in-progress` — flagging because "the learner touched this lab at
   some point" is an equally defensible reading that would keep it
   `in-progress` forever until finished.
2. **The dashboard's "most recent postmortems" has no defined sort order
   or limit.** This change sorts by `solved_at` descending and caps at 5,
   across all `complete` attempts regardless of `publish` flag (per the
   Postmortem viewer requirement's "regardless of their publish flag").
   A different cap or sort (e.g. by file creation time) would also
   satisfy the literal text.
3. **The spec says "the exact CLI command to start or resume it" without
   defining "resume."** There's no separate `breaklab resume` verb in
   this CLI. This change shows `breaklab start <lab-id>` for a lab that
   isn't the currently active one, and `breaklab check` for the lab that
   is — "resume" meaning "the next useful command," not a literal
   subcommand name.

## Open Questions

None — every ambiguity found had a specific, documented resolution
above.
