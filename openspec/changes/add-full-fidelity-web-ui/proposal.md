## Why

A set of Stitch-generated mockups (`breaklab-design/`) proposes a denser,
more "live incident dashboard" look for the local web UI (`breaklab
serve`) than it has today: real Docker engine/resource telemetry, a
live log tail on the active lab, ticket IDs, progressive-hint framing,
and training-velocity analytics, all under a new dark zinc/JetBrains-Mono
design system. The prior restyle
(`openspec/changes/archive/2026-10-18-restyle-web-ui`) deliberately kept
"no new routes, no new template data" — this change intentionally lifts
that constraint for the local UI so its screens can show real data the
app already has the means to compute or query, not just new colors.

## What Changes

- Add five read-only `internal/docker` methods (`Version`, `Stats`,
  `Logs`, `LogsFollow`, `DiskUsage`), each going through the existing
  argument-slice `docker()` exec wrapper — no shell, no new mutation
  capability.
- Add matching injected funcs on `internal/web.Config`
  (`DockerVersion`, `ContainerStats`, `ContainerLogs`,
  `ContainerLogsFollow`, `DiskUsage`, plus a local `Started time.Time`),
  following the existing `DockerStatus StatusFunc` pattern: short
  timeout, session-only container name, degrade to "unknown"/"—"
  rather than failing the page.
- Add one new route: `GET /labs/{id}/logs/stream` (Server-Sent Events),
  streaming the active session's container log tail. Read-only, scoped
  to the active session's own lab/container exactly like the existing
  Topology route, and torn down when the request context cancels.
- Extend the Challenge browser with `q` (search) and `sort` query
  params and a random-launch action — still a server-rendered GET/POST
  flow, no SPA, no JS bundler.
- Add view-layer-only computations with no new persisted fields:
  per-attempt ticket IDs (`TKT-XXXX`, formatted from the existing
  attempt index), a cosmetic hint-cost display, average resolution
  time, postmortem draft/published counts, and training-velocity /
  difficulty-curve / median-solve-time aggregates — all derived from
  `progress.json` and the catalog already on disk.
- Add minimal fenced-code-block rendering to postmortem display (plain
  monospace blocks, `+`/`-` coloring for ```diff fences) — hand-rolled,
  no new markdown dependency.
- Rebuild `templates/web/*.html` and the shared CSS token block
  (`templates/web/style.css`, `templates/site/style.css`) to the new
  "Pedagogical Clarity" design system from
  `breaklab-design/pedagogical_clarity/DESIGN.md`: dark zinc palette
  with light-mode equivalents via `prefers-color-scheme` (no toggle,
  matching the existing `web-ui-design` requirement), JetBrains Mono
  vendored locally for code/labels/tickets, prose on the system sans
  stack.
- Explicitly out of scope (recorded so they aren't silently dropped):
  per-lab "terminal reproducer steps" (hand-authored content, not a UI
  feature), a real hint-cost scoring economy (cost is cosmetic only —
  nothing is deducted from anything), fabricated capacity numbers (no
  invented image-cache quota or host-RAM-vs-cgroup display beyond what
  `docker stats`/`docker system df` actually report), a ⌘K command
  palette, and a fake git SHA (real build revision via
  `runtime/debug.ReadBuildInfo()` instead, blank when unavailable).
- Public site (`templates/site/*`) gets only the shared token/typography
  changes — none of the live Docker-derived panels, since it has no
  Docker to query and must stay fully offline-capable.

## Capabilities

### New Capabilities

(none — all changes extend existing capabilities)

### Modified Capabilities

- `web-ui`: adds the SSE log-stream route, challenge-browser
  search/sort/random-launch, and new Dashboard/Roadmap/Challenge/Lab
  Detail/Postmortem view fields (ticket IDs, resource stats, diagnostic
  output, hint-cost display, training-velocity analytics, postmortem
  code-block rendering), while preserving the existing no-variant-ID
  and no-unrevealed-hint-text leakage requirements.
- `web-ui-design`: adds the "Pedagogical Clarity" token values (color,
  type scale, spacing/radius rules) and a locally-vendored JetBrains
  Mono font, replacing the current token values while keeping the
  existing shared-tokens / light-dark / status-never-color-only
  requirements unchanged in shape.

## Impact

- `internal/docker/docker.go` (+tests): five new read-only methods.
- `internal/web/server.go`: `Config` gains new injected funcs and
  `Started`; `cmd/breaklab`'s serve wiring passes the real
  implementations.
- `internal/web/{dashboard,roadmap,challenges,lab,postmortem}.go`
  (+tests): new view fields, new aggregates, new SSE handler, new
  query-param handling.
- `internal/postmortem/`: small fenced-code-block helper for rendering.
- `templates/web/*.html`, `templates/web/style.css`,
  `templates/site/style.css`: full visual rebuild plus a vendored
  JetBrains Mono asset under `templates/web/fonts/`.
- No changes to `labs/*/lab.yaml`, `progress/progress.json`'s schema,
  or any CLI command — this is local-web-UI-only.
