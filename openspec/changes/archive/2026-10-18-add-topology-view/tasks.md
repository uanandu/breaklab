## 1. Docker status lookup

- [x] 1.1 Add `(*docker.Docker) Status(ctx, name) (status, image string, err error)` to `internal/docker/docker.go`, built like `Exists` (`docker container inspect <name>` via `os/exec` argument slice, `validName` guard), parsing the container's running state and image reference
- [x] 1.2 Give the call a short context timeout (e.g. 2s) at the call site so an unreachable daemon degrades instead of hanging the request, and verify with a unit test using `docker.NewWithExec` that a slow/erroring exec surfaces as an error the caller can treat as "unknown" — timeout added at the call site in `internal/web/topology.go` (task 2.2); `TestStatus`'s "exec error" case confirms `Status` itself surfaces the error rather than swallowing it
- [x] 1.3 Unit test `Status` with `NewWithExec` fixtures for: running container, stopped container, nonexistent container, and a simulated exec error — `internal/docker/docker_test.go`'s `TestStatus`, plus an invalid-name case

## 2. Topology view-model and route

- [x] 2.1 Add `internal/web/topology.go` with `TopologyView` (Host info, `Containers []ContainerNodeView` — slice of one today — each with `Ports []PortNodeView`) and a `buildTopology(cat *catalog.Catalog, session *runner.Session, status func(name string) (string, string, error))` function so Docker lookups are injectable in tests — `status` is a package-level `StatusFunc` type (`ctx` included, per task 1.2's call-site timeout) rather than an inline func type, matching `OpenBrowserFunc`'s existing pattern
- [x] 2.2 Implement `topologyHandler`: load session via `runner.LoadSession` (same pattern as `dashboardHandler`/`labHandler`), load the lab's catalog entry for its declared `Environment.Ports`, call the Docker status lookup only with `session.Container`, and render `topology.html` — the 2s timeout from task 1.2 is applied here via `context.WithTimeout(r.Context(), dockerStatusTimeout)`
- [x] 2.3 Verify the container name passed to Docker is never read from the request (`r.URL.Query()`, form values, or path) — only from the loaded session — and add a test asserting this (e.g. a request with a crafted `?container=...` query param has no effect on which container is inspected) — `TestTopologyHandlerIgnoresContainerFromRequest`
- [x] 2.4 Verify no variant id, unused hint text, or variants/solutions content is reachable from the new route or view-model, matching the existing lab-detail rule — `TopologyView`/`ContainerNodeView` carry no variant or hint field at all (structurally unreachable), plus `TestTopologyHandlerNeverLeaksVariantID` guards against a future regression
- [x] 2.5 Register the `/topology` route in `internal/web/server.go` alongside the existing routes — registered in `internal/web/handlers.go`'s `routes()`, which is where every other route actually lives (`server.go` holds `Serve`/`Config`); wiring the real `*docker.Docker` into `Config.DockerStatus` happens in `cmd/breaklab/serve.go`

## 3. Template and nav

- [x] 3.1 Create `templates/web/topology.html`: host node → container node(s) → port nodes, server-rendered with `html/template` ranges and CSS/SVG for connectors, no JS — used bordered node boxes joined by muted arrow glyphs (`&rarr;`) via CSS flexbox rather than inline SVG lines; simpler, same "plain CSS flexbox/grid ... or CSS border connectors" latitude design.md left open
- [x] 3.2 Render the container detail (name, image, running/unknown status, `docker exec -it <container> bash` command) as an in-page section, linked to from the container node (anchor link, no HTMX fragment route)
- [x] 3.3 Render each port node's host:container mapping, with a best-effort "open" link to `http://localhost:<host-port>`
- [x] 3.4 Add the no-active-lab empty state, matching the dashboard's existing copy/pattern (prompt to start a lab from the roadmap or challenge browser)
- [x] 3.5 Add a "Topology" link to the shared nav across all `templates/web/*.html` pages

## 4. Verification

- [x] 4.1 `go test ./internal/docker/... ./internal/web/...` covering: active lab with container running, active lab with Docker unreachable (status "unknown"), and no active lab — all pass; also extended the existing `TestRouteTableIsReadOnlyAndBounded` security test (internal/web/security_test.go) to cover `/topology` (path-traversal-via-query-param rejection, non-GET methods 405), since this change's own spec requirement extends that same guarantee to Docker-backed routes
- [x] 4.2 Manually run `breaklab start web-down`, open `/topology`, confirm the container node, its port, and the exec command render, and that clicking through shows the detail section — done against a real Docker container in an isolated scratch working dir, verified visually in Chrome: diagram, `badge-running`, image, exec command, port mapping, and the anchor-link click-through to the container detail section all rendered correctly
- [x] 4.3 Manually stop the Docker daemon (or rename the container out from under the session) and confirm `/topology` still renders with status "unknown" instead of a 500 — removed the container out from under the active session (`docker rm -f`); page still returned 200 with the container shown as "unknown", verified both via curl and visually in Chrome
- [x] 4.4 Run `gofmt -l .`, `go vet ./...`, and `go test ./...` — all clean
