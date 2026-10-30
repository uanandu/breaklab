## 1. internal/docker additions

- [x] 1.1 Add `Version(ctx) (string, error)` (`docker version --format
      '{{.Server.Version}}'`) and a unit test using the existing
      `fake(ExecFunc)` pattern in `docker_test.go`
- [x] 1.2 Add `Stats(ctx, container string) (Stats, error)` (`docker
      stats --no-stream --format '{{json .}}' <container>`), validating
      `container` the same way `validName` already does, parsing
      CPU %/mem used/mem limit; unit test covers a valid-JSON fixture
      and a malformed-output case returning an error
- [x] 1.3 Add `Logs(ctx, container string, tail int) ([]byte, error)`
      (`docker logs --tail N <container>`); unit test via the fake exec
- [x] 1.4 Add `LogsFollow(ctx, container string) (io.ReadCloser, error)`
      (`docker logs -f --tail 20 <container>` via a combined-output
      `io.Pipe`, since `cmd.StdoutPipe()` alone can't merge stderr),
      whose returned `Close()` kills the process if still running; unit
      test asserts `Close()` terminates the underlying process (e.g.
      using a short-lived fake command and checking it's no longer
      running after `Close`)
- [x] 1.5 Add `DiskUsage(ctx) (DiskUsage, error)` (`docker system df
      --format '{{json .}}'`); unit test via the fake exec
- [x] 1.6 `go test ./internal/docker/...` passes

## 2. internal/web Config wiring

- [x] 2.1 Add `DockerVersion`, `ContainerStats`, `ContainerLogs`,
      `ContainerLogsFollow`, `DiskUsage` func fields to `Config` in
      `server.go`, each nil-safe the same way `DockerStatus` already is
- [x] 2.2 Add `Config.Started time.Time`, set once when `Serve()` starts
- [x] 2.3 Wire the five new funcs to the real `*docker.Docker` methods
      in `cmd/breaklab`'s serve wiring, next to where `DockerStatus` is
      wired today
- [x] 2.4 `go build ./...` succeeds with the new wiring in place

## 3. View-layer aggregates (pure functions, no Docker/filesystem)

- [x] 3.1 Add `ticketID(attemptIndex int) string` and use it from every
      call site that already has an attempt index (dashboard active-lab
      + recent list, roadmap/challenges rows, lab detail, postmortem
      header — wired in Group 5)
- [x] 3.2 Add `hintCost(hintIndex int) int` (`5*(i+1)`); unit test
- [x] 3.3 Add an average-resolution-time aggregate over solved/complete
      attempts (`SolvedAt - StartedAt`); unit test with a mix of solved,
      active, and abandoned attempts confirms only solved/complete ones
      count
- [x] 3.4 Add postmortem draft/published count aggregates from
      `pf.Attempts`; unit test
- [x] 3.5 Add training-velocity, difficulty-curve, and median-solve-time
      aggregates over `*catalog.Catalog` + `[]progress.Attempt`; unit
      tests for each, including the zero-attempts case
- [x] 3.6 `go test ./internal/web/...` passes for this group in
      isolation (new aggregate tests only, handlers not yet touched)

## 4. internal/postmortem fenced-code rendering

- [x] 4.1 Add a fence-splitting function that turns a section's text
      into an ordered list of paragraph/code-block segments, tagging
      ` ```diff ` fences distinctly from plain fences; unit tests cover
      no-fence text, a plain fence, a diff fence, and an unterminated
      fence
- [x] 4.2 Update `SectionView`/`buildPostmortemPage` in
      `internal/web/postmortem.go` to carry the new segment list instead
      of (or alongside) `Paragraphs`
- [x] 4.3 `go test ./internal/postmortem/... ./internal/web/...` passes

## 5. Handlers: dashboard, roadmap, challenges, lab detail

- [x] 5.1 Extend `buildDashboard`/`DashboardView` with Docker engine
      version, server uptime, extended active-lab fields (ticket, ports,
      target scope, hints-unlocked count, log tail), and the new stat
      tiles (avg resolution time, postmortem drafts/published); test
      covers the Docker-unreachable degrade-to-"—" path
- [x] 5.2 Extend `labRow`/`LabRowView` (shared by roadmap and
      challenges) with the ticket identifier for labs with a recorded
      attempt; test (also wired into the postmortem page header, the
      remaining "every call site" from 3.1 — 3.1 is now fully done)
- [x] 5.3 Extend `buildChallenges` with `q` (search) and `sort` query
      params, the active-incident banner, and the training-velocity
      footer stats; tests for search matching, each sort order, and
      filters-plus-search combined
- [x] 5.4 Add `POST /challenges/random`: filters the catalog the same
      way `buildChallenges` does (shared `filterLabs` helper), picks one
      lab via `math/rand/v2`, redirects (303) to its detail page; test
      covers both a matching set and the empty-result-set case (redirect
      to `/challenges`, no panic)
- [x] 5.5 Extend `buildLabDetail`/`LabDetailView` with the ticket
      identifier, log tail (active only), and per-hint cost labels
      (locked hints only — never reveal text); test explicitly asserts
      no variant id or unrevealed hint text appears in the rendered view
      struct (`HintSlotView` structurally carries no text field at all,
      only Revealed/Cost)
- [x] 5.6 Add `GET /labs/{id}/logs/stream`: SSE handler, 404 unless
      `id == session.Lab`, streams `ContainerLogsFollow` lines (its own
      `docker logs -f --tail 20` already opens with the last 20 lines,
      so no separate upfront `ContainerLogs` call is needed — a
      redundant second docker invocation would have duplicated the same
      lines), closes the `ReadCloser` (killing the subprocess) on
      `r.Context().Done()`; test covers the wrong-lab-id 404 case and
      that cancelling the request context triggers `Close` on a fake
      `ReadCloser`
- [x] 5.7 `go test ./internal/web/...` passes in full

## 6. Templates + CSS

- [x] 6.1 Vendor JetBrains Mono (OFL) under `templates/web/fonts/` and
      HTMX core + its `sse` extension under `templates/web/js/`; both
      referenced locally, no CDN URL anywhere in the templates (served
      via a new `staticAssetHandler`; a dedicated test confirms the
      active lab page's script tags are vendored-local only, and the
      pre-existing "no `<script>` tag on any page" test now scopes that
      rule to every page except an active lab's own live-tail script)
- [x] 6.2 Port `breaklab-design/pedagogical_clarity/DESIGN.md`'s token
      values into the shared custom-property block in
      `templates/web/style.css`, with light-mode equivalents under the
      existing `prefers-color-scheme: light` block
- [x] 6.3 Rewrite `templates/web/dashboard.html` for the new telemetry
      fields and visual system; status/difficulty badges keep
      color+text/shape per the existing `web-ui-design` requirement
- [x] 6.4 Rewrite `templates/web/roadmap.html` (stage cards, ticket ids,
      progress bars, real build revision footer via
      `runtime/debug.ReadBuildInfo()`)
- [x] 6.5 Rewrite `templates/web/challenges.html` (search box, sort
      control, topic chips, active-incident banner, random-launch
      button, footer stat tiles) — also fixed a bug caught while wiring
      this up: the random-launch button submits as a POST body, not a
      URL query string, so the handler now reads `r.Form` (parsed from
      both) instead of `r.URL.Query()` alone; regression test added
- [x] 6.6 Rewrite `templates/web/lab.html` with the `hx-ext="sse"` log
      panel, ticket header, locked-hint cost labels (no text), and
      removal of the previously-planned per-lab reproducer-steps section
      (explicitly descoped)
- [x] 6.7 Rewrite `templates/web/postmortem.html` to render the new
      paragraph/code-block segment list, with diff-tagged blocks
      colored and prefixed `+`/`-`
- [ ] 6.8 Manual pass: open each of the five pages with a dark-mode and
      a light-mode browser profile, confirm no flash of unstyled content
      and no element relying on color alone for status/difficulty

## 7. Public site token pull-through

- [x] 7.1 Port only the shared color/type/spacing token values (not any
      live-data markup) into `templates/site/style.css`'s matching
      custom-property block, plus the same vendored-font `@font-face`
      pair (fonts copied to `templates/site/fonts/`); `internal/site.Build`
      gained a `copyFonts` step alongside its existing `copyStyle`
- [x] 7.2 `breaklab publish` (or the equivalent test) still produces a
      `site/index.html` that opens correctly from disk with no network
      connection, per the existing progress-site offline requirement

## 8. Verification

- [x] 8.1 `go test ./...` passes
- [x] 8.2 `go vet ./...` passes (gofmt -l also clean)
- [x] 8.3 `govulncheck ./...` passes: "No vulnerabilities found."
- [x] 8.4 Manual run, in an isolated copy of the repo (never the real
      `progress/progress.json`/`.breaklab/session.json` — those hold
      real recorded history): built the binary, started `web-down`,
      ran `breaklab serve`, and drove it with both `curl` and a real
      Chrome tab. Confirmed in-browser: the dashboard's real Docker
      version/uptime/ticket-id/diagnostic-output tiles, the challenge
      browser's incident banner/search/filters/training-velocity/
      difficulty-curve/sandbox-image-cache footer (real `docker system
      df` output), and the roadmap's per-lab ticket ids. The live log
      tail was verified genuinely live: swapped the lab's container for
      one that prints an incrementing counter, and watched new lines
      arrive in the browser via the real SSE connection (not simulated).
      This also caught and fixed two real bugs (see below). The
      diff-block postmortem rendering was verified via its unit tests
      and the real-template execution test
      (`TestRealTemplatesRenderWithoutScriptTags`'s fixture renders the
      actual `postmortem.html`) rather than a live browser screenshot —
      the manual fixture had no published postmortem with a `diff`
      fence to hand
- [x] 8.5 Verified subprocess cleanup on client disconnect two ways:
      the unit test `TestLabLogsStreamClosesOnClientDisconnect`, and
      manually via `curl -N` against the real running server, killed
      mid-stream, confirming via `ps` that no `docker logs -f` process
      for the lab's container lingered afterward

### Bugs found and fixed during manual verification

- **Challenge-browser random-launch read the wrong request data.** The
  "Launch random" button submits as a POST body (`formmethod="post"`),
  but `challengesRandomHandler` read `r.URL.Query()`, which is always
  empty for a POST body submission — every active filter was silently
  ignored. Fixed by parsing `r.Form` (covers both URL query and POST
  body) instead; added `TestChallengesRandomHonorsFormBodyFilters` to
  pin this down.
- **The live log tail inserted unescaped, unbroken HTML.** htmx's
  `sse-swap` inserts each event's `data:` payload as HTML
  (`innerHTML`-style), not text, so (1) a container's own log output —
  attacker-influenceable content, since it's whatever the broken
  process prints — could inject markup/script into the viewing browser,
  and (2) with no block-level wrapper, successive appended lines ran
  together with no line break. Fixed by having
  `labLogsStreamHandler` HTML-escape each line and wrap it in a
  `<span class="log-line">` (the already-defined `display: block` log
  line class); added `TestLabLogsStreamEscapesLogContent` and confirmed
  the line-break fix visually in a real browser tab.
- [ ] 8.6 Manual check: confirm a locked hint never renders its text
      anywhere in the rendered HTML (view-source), and that no variant
      id appears anywhere on the lab detail or dashboard pages
