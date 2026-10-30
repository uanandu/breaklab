# Tasks

## 1. `internal/progress`: shared per-lab helpers

- [x] 1.1 Promote `internal/site`'s private "latest complete attempt per lab" lookup into exported `progress.LatestCompleteAttempt(pf *ProgressFile, labID string) (Attempt, bool)` and `progress.LatestCompleteAttempts(pf *ProgressFile) map[string]Attempt` (keyed by lab id); refactor `internal/site/view.go`'s `BuildIndex` to call the latter instead of its own private copy, which is deleted. Verify the existing `internal/site` test suite passes unchanged, plus new `internal/progress` unit tests: a lab with two `complete` attempts returns the later one; a lab with no `complete` attempts is absent from the map; an empty `ProgressFile` returns an empty map.
- [x] 1.2 Implement `progress.ApplyStatus(cat *catalog.Catalog, pf *ProgressFile)`, mutating each `Lab.Status` to `"complete"` (any `complete` attempt), `"in-progress"` (any `active` or `solved` attempt and no `complete` one), or `"available"` otherwise — including a lab whose only attempts are all `abandoned` (design.md Decision 2). Verify with unit tests covering all four input shapes (complete, active-only, solved-only, abandoned-only) plus a lab with no attempts at all, and a lab with both an abandoned and a later active attempt (still `"in-progress"`, not reset).

## 2. Fix `breaklab list`'s status column

- [x] 2.1 Update `cmd/breaklab/list.go` to load `progress/progress.json` (a missing file is fine — `progress.Load` already returns an empty `ProgressFile`, no error) and call `progress.ApplyStatus` on the loaded catalog before printing. Verify with a test asserting a lab with a `complete` attempt in a fixture `progress.json` prints `complete` via `breaklab list`'s output, and confirm by hand that `breaklab list` still runs cleanly in a fresh repo with no `progress/progress.json` at all.

## 3. `internal/web`: server scaffolding and dashboard

- [x] 3.1 Implement the server core in `internal/web`: `Config` (labs dir, roadmap path, session path, progress path, templates dir, host, port) and `Serve(ctx context.Context, cfg Config) error` — `net.Listen` first (so a bound port produces a clear "port N is already in use" error, not a generic bind failure), then `http.Serve` on that listener with graceful shutdown when `ctx` is cancelled, plus a `/style.css` route serving the fixed `templates/web/style.css` path (never request-derived). Verify with unit tests: starting on an already-bound port returns the specific port-in-use error; cancelling `ctx` shuts the server down cleanly (bind to port 0 for the test, dial it, cancel, confirm the listener closes).
- [x] 3.2 Implement the default-browser opener as an injectable `OpenBrowserFunc func(url string) error` (default: `xdg-open`/`open`/`start` by `runtime.GOOS`, run via `exec.Command` directly, never a shell), wired so it's called only when the bind host is a loopback address, and a one-time warning is printed to stdout/stderr when it isn't. Verify with a unit test injecting a fake opener: called with the right URL for `127.0.0.1`, not called for `0.0.0.0`, with the warning text present only in the non-loopback case.
- [x] 3.3 Implement the dashboard view and `GET /` route: streak (`progress.Streak`), totals (`LabsCompleted` = `len(progress.LatestCompleteAttempts(pf))`, `VariantsSeen` = distinct lab+variant pairs across every attempt, matching design.md Decision 6's complete-only definition), the active lab's elapsed time and hints used (via `runner.LoadSession` — live `HintsUsed`, elapsed from the attempt's `StartedAt`) when one is running, and the five most recently solved postmortems (by `SolvedAt` descending, from `progress.LatestCompleteAttempts`' values, regardless of `publish`). Verify with unit tests: no active session shows totals/streak and a start-a-lab prompt only; an active session shows elapsed minutes, hints used, and a `breaklab check` reminder; more than five completed labs shows only the five most recent.

## 4. Roadmap and challenge browser

- [x] 4.1 Implement the roadmap view and `GET /roadmap` route: load the catalog and `progress.json`, call `progress.ApplyStatus`, and render every stage in `roadmap.yaml` order with each lab's status, difficulty, and estimated minutes, linking a `complete` lab to its postmortem viewer page when it has one. Verify with unit tests: a fixture with one lab in each of the three statuses renders all three correctly, in roadmap stage order; a `complete` lab without a recorded postmortem path shows no link.
- [x] 4.2 Implement the challenge browser view and `GET /challenges` route: the same per-lab status/difficulty data across every stage, filterable via `?stage=&difficulty=&status=` on a plain `<form method="get">` (no JS). Verify with unit tests: filtering by stage, by difficulty, and by status each narrow the list correctly; an unrecognized filter value yields an empty result rather than an error; no query params (or a link clearing them) returns the full list.

## 5. Lab detail and postmortem viewer

- [x] 5.1 Implement the lab detail view and `GET /labs/{id}` route: ticket text (read from `labs/<lab.ID>/TICKET.md` using the matched catalog lab's own validated `ID`, never the raw path value), status, a hints-used count (live from the session when this is the active lab, otherwise the latest attempt's persisted `HintsUsed`, otherwise 0), and the CLI command to run (`breaklab start <id>` normally, `breaklab check` when this lab is the active one) — never a variant id or any hint text, solved or not (design.md Decision 4). Verify with unit tests: an unsolved lab's response contains none of its own hint strings or its variant id; the shown command differs correctly between the active lab and any other; an unknown lab id returns 404.
- [x] 5.2 Implement the postmortem viewer and `GET /postmortems/{slug}` route: build a slug-to-path map from every `complete` attempt's `Postmortem` field (never by joining the request slug into a filesystem path), and render the matched file's front matter and all five sections — reusing `postmortem.SplitFrontMatter`/`ParseFrontMatter`/`SplitIntoSections`/`StripComments` — regardless of its `publish` flag. Verify with unit tests: a `complete` attempt's postmortem renders fully even with `publish: false`; an unknown or path-traversal-shaped slug (e.g. containing `..` or `/`) returns 404 without reading any file outside the known set.

## 6. Templates

- [x] 6.1 Write `templates/web/style.css` and the five page templates (`dashboard.html`, `roadmap.html`, `challenges.html`, `lab.html`, `postmortem.html`) as self-contained `html/template` documents (matching `templates/site/`'s existing convention — no shared layout abstraction, no JS). Verify with a unit test asserting none of the five rendered pages contains a `<script` tag, confirming each view's own tests (tasks 3.3, 4.1, 4.2, 5.1, 5.2) render through these real template files rather than stubs.

## 7. CLI command

- [x] 7.1 Add `breaklab serve [--host] [--port]` to `cmd/breaklab/` (default `127.0.0.1`/`8787`), printing the URL before calling `internal/web.Serve`, and register it in `commands`/`commandOrder` in `cmd/breaklab/main.go`. Verify with a CLI-level test asserting `breaklab serve --help` exits 0 and is listed in `commandOrder` (without actually binding a port in that test).
- [x] 7.2 Verify by hand against a real lab: start `web-down`, run `breaklab serve` against a test port in the background, and `curl` `/`, `/roadmap`, `/challenges`, `/labs/web-down`, and `/style.css`, confirming 200s and that the lab detail response contains none of `web-down`'s hint text or its active variant id; solve the lab, create its postmortem, and confirm `/postmortems/<slug>` renders it even before setting `publish: true`; run `breaklab list` and confirm its status column now reflects progress instead of always showing `available`; stop the server and the lab.

## 8. Docs consistency check

- [x] 8.1 Re-read `docs/deploy-raspberry-pi.md`'s claims about `breaklab serve` (default port 8787, `--host`/`--port` flags, the port-in-use error naming the port, the one-time non-loopback warning) against the implementation from tasks 3.1–3.2 and 7.1, and update the doc only if a real mismatch is found — expect none, since the doc was written to match this change's intended behavior.

## Workflow follow-up

- Run `gofmt -l` / `go vet` and the full `go test ./...` suite before archiving.
- Run `openspec validate add-local-web-ui --strict` and fix any findings.
- Confirm `templates/web/` is trackable (not gitignored).
- Archive the change with `/opsx:archive` once the above pass and the user has reviewed it.
