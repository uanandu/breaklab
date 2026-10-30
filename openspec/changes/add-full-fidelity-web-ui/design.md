## Context

See `proposal.md` for motivation. Current state relevant to the
approach:

- `internal/web` already has one established pattern for touching
  Docker from a request: `StatusFunc` on `Config` (`topology.go`),
  called with a 2s timeout, degraded to "unknown" on error, and fed
  only the session-recorded container name — never request input.
  Every new Docker-backed piece of data in this change reuses that
  exact shape.
- `internal/docker.Docker` wraps the `docker` CLI via `os/exec` with
  argument slices (`docker.go`'s `d.docker(ctx, args...)`); no shell,
  no string interpolation. New methods extend this file rather than
  introducing a Docker SDK dependency.
- Views are built by small pure functions (`buildDashboard`,
  `buildChallenges`, `buildLabDetail`, `buildTopology`) that take
  already-loaded data (`*catalog.Catalog`, `*progress.ProgressFile`,
  `*runner.Session`) and return a template-ready struct. This change
  keeps that split: aggregates and formatting are pure functions,
  Docker/filesystem reads happen only in the handler and get passed in.
- The project's declared stack includes HTMX, but no template actually
  uses it yet — every current route is a plain form POST + full-page
  GET. This change is the first to use it, for exactly one thing: the
  live log tail.
- `progress.json` and `lab.yaml` schemas are untouched by this change.
  Every new display value (ticket id, hint cost, training-velocity
  stats) is computed from data already on disk.

## Goals / Non-Goals

**Goals:**
- Real Docker/system telemetry (engine version, container stats, log
  tail, disk usage) surfaced through the same safe, degrade-gracefully
  pattern Topology already established.
- One live-updating element (the lab-detail log tail) without adding a
  JS build step or client-side framework.
- All cosmetic/derived values (ticket ids, hint cost, training-velocity
  analytics) computed losslessly from existing data — zero new
  persisted fields.

**Non-Goals:**
- No scoring/points system — hint cost is a label, not a stored value.
- No per-lab authored "reproduction steps" content pipeline.
- No change to `progress.json`/`lab.yaml` schema or to any CLI command.
- No markdown library — postmortem code-block handling is a small
  hand-rolled fence detector, not general markdown rendering.
- No auth, no change to the loopback-default/`--host` opt-in model.

## Decisions

### Docker telemetry: extend `internal/docker`, inject via `Config`, same degrade-on-error contract as `StatusFunc`
New methods `Version`, `Stats`, `Logs`, `LogsFollow`, `DiskUsage` on
`*docker.Docker`, each a thin wrapper around one `docker` subcommand
with `--format` for machine-parseable output (`{{json .}}` where
available). `internal/web.Config` gets matching func fields
(`DockerVersion`, `ContainerStats`, `ContainerLogs`,
`ContainerLogsFollow`, `DiskUsage`), wired to the real `*docker.Docker`
in `cmd/breaklab`'s serve wiring — identical to how `DockerStatus` is
wired today. `internal/web` never imports `internal/docker` directly,
preserving the existing decoupling.

Every call site uses a short `context.WithTimeout` (reuse
`dockerStatusTimeout` or a sibling constant) and treats any error as
"no data" rather than a request failure: dashboard/lab-detail render
with the affected tile showing "—"/"unknown", never a 500.

Alternative considered: pull in the Docker Go SDK
(`github.com/docker/docker/client`) for structured stats instead of
parsing CLI `--format` output. Rejected — it's a much larger dependency
surface for a single-binary tool that otherwise has almost no
third-party deps, and the existing `Build`/`Run`/`Status` methods
already establish CLI-shelling as this project's Docker integration
style; staying consistent beats marginally cleaner parsing.

### Container name/ownership for every new Docker call: session-only, same rule as Topology
`Stats`, `Logs`, and `LogsFollow` all take a container name; every
caller in `internal/web` passes `session.Container` — set and validated
by `runner.LoadSession` against `^[a-z0-9-]+$` and the exact
`containerName(lab)` format — never a path or query value. This is the
same guarantee the existing `web-ui` spec's "Web UI attack surface
covers Docker-backed routes" requirement already states in
lab-id-agnostic terms ("any local web UI route that reads live status
from Docker... including the Topology view"), so no change to that
requirement's text is needed — the new routes already fall under it.

### Live log tail: Server-Sent Events via HTMX's `sse` extension, not hand-rolled `EventSource` JS
`GET /labs/{id}/logs/stream` is an SSE endpoint. The lab-detail
template uses `hx-ext="sse"` / `sse-connect` / `sse-swap` (htmx core +
its small `sse.js` extension, vendored locally under
`templates/web/js/`) to append each line to a `<pre>` panel, rather
than writing a bespoke `EventSource` handler. This is a better fit than
ad hoc JS because HTMX is already the project's declared (if
previously unused) answer to "no SPA framework, no JS build step" —
using it here is the first real instance of that stack decision rather
than a second, competing one.

Handler side: on connect, write the `ContainerLogs` tail (e.g. last 20
lines) as the first SSE frames, then stream `ContainerLogsFollow`'s
`io.ReadCloser` line-by-line as further `event: message\ndata: ...`
frames, flushing after each (`http.Flusher`). The handler selects on
`r.Context().Done()` and the read loop; on cancellation it closes the
`ReadCloser`, which must in turn terminate the underlying `docker logs
-f` subprocess (not just stop being read) — `LogsFollow` returns a
`io.ReadCloser` backed by `cmd.StdoutPipe()` whose `Close` calls
`cmd.Process.Kill()` if the process hasn't exited, so an abandoned
connection can't leak a running `docker logs -f` process.

Route guard: 404 unless `r.PathValue("id") == session.Lab` — mirrors
`buildTopology`'s "no active session → no diagram" and the lab-detail
handler's existing `findLab`/`Active` check; this is also where the
spec's "stream scoped to the active lab only" scenario is enforced.

Alternative considered: plain polling (`hx-get` + `hx-trigger="every
2s"`) against a route that returns the latest tail each time. Simpler,
but strictly worse here — it would need a 2s backend `docker logs
--tail N` shell-out per poll per open tab, is less "live" than the
mockup asks for, and SSE is barely more code once `LogsFollow` exists
for the dashboard's one-shot tail anyway.

### Ticket IDs, hint cost, and analytics are pure view-layer functions — no new storage
`ticketID(attemptIndex int) string` → `fmt.Sprintf("TKT-%04d",
1000+attemptIndex)`. Every call site already has the attempt index in
hand (`Session.AttemptIndex` for the active lab; the position within
`pf.Attempts` for any other attempt via the existing `latestAttempt`/
`LatestCompleteAttempts` helpers), so this needs no new field on
`Attempt` and no counter to persist or race against. Similarly,
`hintCost(hintIndex int) int` is `5 * (hintIndex + 1)`, and the
training-velocity/difficulty-curve/median-solve-time/avg-resolution-
time aggregates are computed by new pure functions over
`[]progress.Attempt` and `*catalog.Catalog`, parallel to the existing
`progress.Streak`/`progress.LatestCompleteAttempts`.

Alternative considered: persist a `Ticket int` field on `Attempt` at
creation time. Rejected — it's strictly redundant with the attempt's
own position in the slice, adds a migration concern for existing
`progress.json` files with no `Ticket` field, and buys nothing a pure
function doesn't already give.

### Postmortem code blocks: hand-rolled fence detector, not a markdown dependency
`internal/postmortem` gains a small function that walks a section's
text, splits on ` ``` ` fences, and returns an ordered list of
paragraph/code-block segments instead of `SectionView.Paragraphs`
alone. A fence tagged `diff` marks its lines as diff-colored in the
template (`+`/`-` prefix already carries the semantic meaning; color is
additive, per the existing `web-ui-design` "never color-only" spirit).
This is deliberately not general markdown (no links, no nested lists,
no tables) — postmortems are validated against five fixed sections of
prose plus optional fenced code, and that's the only structure this
change needs to render.

Alternative considered: add `github.com/yuin/goldmark`. Rejected for
the same reason the Docker SDK was rejected — this project has almost
no third-party dependencies by design, and the actual rendering need
(paragraphs + optional fenced/diff blocks) is small enough that a
~40-line hand-rolled splitter is less risk than a new dependency to
track through `govulncheck`.

### Typography: vendor JetBrains Mono locally; keep prose on the system sans stack
`templates/web/fonts/JetBrainsMono[-weights].woff2` (OFL-licensed,
vendored as static files, served by the existing `/style.css`-adjacent
static handling — no CDN `@import`). Prose stays on
`ui-sans-serif, system-ui, ...` rather than also vendoring Geist, to
keep the asset footprint and license surface small; this is a visual
approximation of the mockup, not pixel-identical. Flagged in
`proposal.md`'s descope list already.

### Challenge search/sort/random-launch stay server-rendered
`q` and `sort` become additional query params on the existing GET
`/challenges` route, handled in `buildChallenges` exactly like
`stage`/`difficulty`/`status`/`tag` are today — no client-side
filtering, no new JS beyond the log-tail's HTMX usage. Random-launch is
a `POST /challenges/random` that re-runs the same filter logic
server-side, picks one row via `math/rand/v2`, and redirects
(`http.StatusSeeOther`) to that lab's existing detail page — it does
not itself start the lab; the user still clicks Launch there, so there
is exactly one place a lab actually starts (unchanged from today).

## Risks / Trade-offs

- **[Risk]** `docker stats --no-stream` and `docker system df` can be
  slow on an already-loaded Pi 5. → Mitigation: same short timeout +
  graceful-degrade contract as every other Docker call; dashboard and
  challenge-browser stat tiles are allowed to show "—" without
  blocking the rest of the page.
- **[Risk]** An SSE connection left open by an abandoned browser tab
  could leak a `docker logs -f` process if `Close()` doesn't actually
  kill it. → Mitigation: `LogsFollow`'s `ReadCloser.Close` explicitly
  kills the process (tested); handler also ties the stream's lifetime
  to `r.Context()`, which `net/http` cancels on client disconnect.
- **[Risk]** Vendoring HTMX + its SSE extension adds two small JS files
  to a project that currently ships none. → Mitigation: both are
  already the project's declared stack choice (see Context), vendored
  locally (no CDN), and used for exactly one feature — this isn't a
  second competing JS strategy, it's the first real use of the
  existing one.
- **[Trade-off]** Ticket ids are derived from attempt position, so if a
  user ever hand-edits `progress.json` to delete an earlier attempt,
  later tickets renumber. Accepted: `progress.json` is already
  documented as user-owned, hand-editable data, and nothing depends on
  a ticket id being stable across such an edit (it's informational
  only, never a lookup key).

## Migration Plan

No data migration — every new field is either computed at render time
or a new optional `Config` func that's `nil`-safe (same pattern as
`DockerStatus`'s existing nil-check, extended to the new funcs). Ships
as one change; rollback is reverting the commit, since nothing persists
a new on-disk shape.
