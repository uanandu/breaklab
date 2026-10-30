## Context

The local web UI already has a consistent pattern for anything
session-dependent (`internal/web/dashboard.go`, `internal/web/lab.go`):
a handler calls `runner.LoadSession(cfg.SessionPath)` to get the active
`*runner.Session` (nil when no lab is running), combines it with
`catalog.LoadCatalog` and `progress` data, and renders a view-model
through `html/template`. No existing route calls into `internal/docker` —
Docker is only ever driven by the CLI (`internal/runner`). See
proposal.md - Why for the motivation; see openspec/specs/web-ui/spec.md
for the existing view requirements this follows the shape of.

v1's architecture is strictly one container per active lab (`labs/<id>/`
has one `Dockerfile`, `runner.Start` runs one `docker run`). Multi-
container labs (`add-compose-labs`) are on the post-v1 roadmap but not
designed yet.

## Goals / Non-Goals

**Goals:**
- A visual, clickable system map of the active lab, built entirely from
  data the app already has (session file, lab catalog metadata) plus one
  new read-only Docker status lookup.
- A data model for the diagram (nodes + edges) that doesn't need to be
  thrown away when multi-container labs arrive — adding more container
  nodes later should be additive, not a rewrite.
- Keep the "no shell execution from a request, no arbitrary path/command
  from a query parameter" rule intact for this new Docker-touching route.

**Non-Goals:**
- No support for multi-container/compose labs now — the diagram renders
  exactly one container node in v1, because that's what exists. The data
  model just doesn't hard-code "exactly one."
- No general-purpose graph/diagramming library, no JS build step, no
  client-side rendering framework — the diagram is server-rendered
  HTML/CSS (and inline SVG for connector lines if needed), consistent
  with the rest of the local web UI.
- No live-updating diagram (no polling/websocket) — the page reflects
  state as of the request, same as every other local UI page; HTMX's
  existing `hx-get` polling idiom could add that later if wanted, but
  it's not required here.
- No historical/past-container view — only the currently active lab, if
  any.

## Decisions

**Model the diagram as a small, explicit node/edge list built server-side,
not a generic graph component.** A `TopologyView` with a `Host` node, one
`Container` node (slice of one today, `[]ContainerNode` so a later
compose-labs change extends it instead of restructuring it), and a
`Ports []PortNode` slice per container. Rendered with nested
`html/template` ranges and plain CSS flexbox/grid for layout, with a thin
inline SVG (or CSS border connectors) for the lines between boxes.
Alternative considered: a JS-based diagram library (e.g. a force-directed
graph) — rejected, it would be the first JS dependency / build step in a
project that has deliberately avoided one (HTMX only), and a one-or-few-
node diagram doesn't need force layout.

**New `docker.Inspect`-style read-only call, reusing the existing
`validName` guard.** Add a method to `internal/docker/docker.go` shaped
like the existing `Exists` (`docker container inspect <name>`), parsing
just status and image from its output, called with the container name
already validated and stored in `session.Container` — never from request
input. This keeps the new Docker touchpoint inside the same package and
validation pattern the CLI already uses, rather than inventing a second
way to shell out.

**Detail panels are server-rendered, not HTMX-fetched fragments.**
Clicking a node navigates to `/topology#container` / renders the detail
inline on the same page load (anchor-link to a section already present in
the response) rather than an HTMX round-trip to a separate fragment
route. Simpler, and avoids adding a route whose sole job is re-deriving
the same session+catalog+docker data the page already computed once.
Alternative considered: an HTMX-swapped detail panel per node — rejected
for v1 as unnecessary indirection when the whole page's data is already
cheap to compute (one session read, one catalog load already cached by
other routes, one `docker inspect`).

**Degrade to "unknown" status on Docker failure rather than failing the
page.** Every other local UI route works entirely from files on disk and
never fails because of an external process. Topology is the first to
depend on Docker being reachable; matching that route's own error
contract (500 on failure) would make the one new Docker-dependent feature
needlessly fragile compared to the rest of the UI. Catching the error and
rendering `status: "unknown"` keeps the page useful (you can still see
what's declared, and get the `docker exec` command) even when the daemon
is down.

## Risks / Trade-offs

- [First local-UI route that calls out to an external process (Docker)
  could add latency or hang the request if the daemon is slow/unreachable]
  → call `docker container inspect` with a short context timeout (e.g. 2s,
  matching the kind of bound the CLI's own Docker calls implicitly get
  from command-line usage) and treat a timeout the same as "unreachable":
  render status unknown, don't block the page.
- [A clickable "open http://localhost:<port>" link assumes the port
  serves HTTP, which isn't declared anywhere in lab.yaml] → label it as a
  best-effort "open" link, not a guarantee; this is a convenience, not a
  correctness claim, and every lab in the Starter labs set (web-down) is
  in fact an HTTP service.
- [Diagram layout for a single container can look sparse/trivial compared
  to what "topology" usually implies] → lean into that honestly: the
  empty-state and single-container view are designed to look complete and
  intentional at one node, not like a placeholder waiting for more nodes.

## Migration Plan

No data migration — purely additive (new route, new template, new nav
link, new read-only Docker method). Rollback is a plain git revert; no
schema or persisted state is introduced.
