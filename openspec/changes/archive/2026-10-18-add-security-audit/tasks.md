# Tasks

## 1. Shared validation helpers

- [x] 1.1 Export `internal/catalog`'s private `validID` as `catalog.ValidID(s string) bool` (same `^[a-z0-9-]+$` pattern, no behavior change), updating its existing callers. Verify the existing `internal/catalog` test suite passes unchanged, plus a unit test covering accepted ids and rejected ones (`..`, `a/b`, `Web_Down`, empty string).
- [x] 1.2 Add `postmortem.Confined(dir, recorded string) (string, error)` (design.md Decision 4): accept `recorded` only if, once cleaned and made absolute, it's a direct child of `dir`'s absolute path, ends in `.md`, and isn't a symlink (`os.Lstat`); return the cleaned path, or an error naming the rejected path. Verify with unit tests: a direct `.md` child passes for both relative and absolute inputs; `dir/../x.md`, `dir/sub/x.md`, `/etc/passwd`, a non-`.md` file, and a symlink inside `dir` are all rejected.

## 2. Close the data-file validation gaps

- [x] 2.1 Make `runner.LoadSession` reject a session whose `lab` or `variant` fails `catalog.ValidID`, or whose `container` isn't exactly `"breaklab-" + lab`, with the same "is corrupt … delete it to clear the session" error shape it already uses (design.md Decision 3). Verify with unit tests: a session written by `Start` still round-trips; sessions with `container: "-f"`, `container: "breaklab-other"`, and `lab: "../x"` are each rejected with an error naming the file; and `Stop` on a tampered session errors without making any Docker call.
- [x] 2.2 In `internal/runner/postmortem.go`, validate the selected attempt's `lab` and `variant` with `catalog.ValidID` before building a filename, and pass the path resolved by the no-path form of `ValidatePostmortem` through `postmortem.Confined(r.PostmortemsDir, …)` before reading it. Verify with unit tests: a `progress.json` attempt with `variant: "../../x"` makes `breaklab postmortem` error and creates no file anywhere; a recorded postmortem path outside `PostmortemsDir` makes no-path `--validate` error without reading it; the existing postmortem tests still pass.
- [x] 2.3 Add `PostmortemsDir` to `site.Config` (default `postmortems` in `cmd/breaklab/publish.go`) and confine `site.LoadPublished`'s reads with `postmortem.Confined`, skipping rejected paths the same way it already skips missing ones. Update the existing site test fixtures to set it. Verify with a unit test: a `complete` attempt whose recorded postmortem points outside the postmortems dir, at a `publish: true` file, contributes no page, link, or text to `site/`; the existing site tests, including "nothing leaks" and "works offline", still pass.
- [x] 2.4 Add `PostmortemsDir` to `web.Config` (default `postmortems` in `cmd/breaklab/serve.go`) and drop any recorded path that fails `postmortem.Confined` from the web UI's slug map, so it's a 404 rather than a read. Update the existing web test fixtures to set it. Verify with a unit test: a `complete` attempt whose recorded postmortem points outside the postmortems dir returns 404 at its slug, and isn't linked from the dashboard or roadmap.

## 3. Process execution

- [x] 3.1 Replace the Windows branch of `web.DefaultOpenBrowser` (`cmd /c start "" <url>`) with `rundll32 url.dll,FileProtocolHandler <url>` (design.md Decision 2). Verify it cross-compiles for `windows/amd64` (and the other three release targets), and that `internal/web`'s tests still pass.
- [x] 3.2 Add a repo-wide audit test package (`internal/audit`, tests only, plus a one-line `doc.go`) with an AST scan of every non-test `.go` file under `internal/` and `cmd/`: fail on any `exec.Command`/`exec.CommandContext` call whose literal arguments name a shell (`sh`, `bash`, `cmd`, `powershell`) together with `-c` or `/c`, or that has an argument built with `+` or `fmt.Sprintf`. Verify it passes on the current tree, and confirm it fails when a throwaway file with `exec.Command("sh", "-c", "x")` is temporarily added (then removed).

## 4. Web UI surface, default bind, and gitignore

- [x] 4.1 Add a test enumerating the web UI's route table: exactly `GET /`, `/roadmap`, `/challenges`, `/labs/{id}`, `/postmortems/{slug}`, and `/style.css` are served; a non-GET method returns 405; and adding a `?path=/etc/passwd`-style query parameter to each route never puts that file's contents in the response. Verify it passes.
- [x] 4.2 Add a CLI-level test that `breaklab serve`'s `--host` flag defaults to `127.0.0.1` (so a non-loopback bind always requires an explicit `--host`). Verify it passes.
- [x] 4.3 In `internal/audit`, add a test that `.gitignore` lists `.breaklab/` and `site/`, and that `git ls-files` (when run inside a git work tree) reports nothing under `.breaklab/` or `site/`. Verify it passes.

## 5. CI

- [x] 5.1 Create `.github/workflows/ci.yml` (design.md Decision 5), triggered on `pull_request` and push to `main`, with `permissions: contents: read` and no `pull_request_target`. Jobs: `gofmt -l` (fail on any output) + `go vet` + `go test ./...`; the `BREAKLAB_INTEGRATION=1` lab integration test; a `GOOS=linux GOARCH=arm64` build; and `go run golang.org/x/vuln/cmd/govulncheck@v1.9.0 ./...`. Verify the YAML parses and those jobs, triggers, and permissions are present.
- [x] 5.2 Pin every `uses:` line in `ci.yml` and `publish.yml` to a full 40-character commit SHA with the tag kept as a trailing comment, resolving each SHA with `git ls-remote https://github.com/actions/<name> "refs/tags/<tag>^{}"` (falling back to the unpeeled ref only for a lightweight tag) and never writing one from memory (design.md Decision 6). Add an `internal/audit` test that fails if any workflow's `uses:` isn't `@<40 hex chars>`, or if any workflow uses `pull_request_target`. Verify the test passes and that each pinned SHA matches what `git ls-remote` returns.

## 6. Audit verification

- [x] 6.1 Run `govulncheck@v1.9.0 ./...` locally, which needs `vuln.go.dev` approved in the sandbox network policy, and confirm it reports no reachable vulnerabilities. If the approval isn't granted, record that this one check is unverified locally and will first run in CI.
  - Network approval was granted. First run (toolchain `go1.26.1`, matching the `go.mod` at the time) found 25 reachable stdlib vulnerabilities, all fixed in later `go1.26.x` patches (e.g. GO-2026-4870, GO-2026-4866, GO-2026-4865). Bumped `go.mod` to `go 1.26.9` (latest `1.26.x` patch); `govulncheck` then reports "No vulnerabilities found." `gofmt`, `go vet`, and `go test ./...` all still pass on `go1.26.9`.
- [x] 6.2 Verify by hand with the real binary: a hand-edited `.breaklab/session.json` with `"container": "-f"` makes `breaklab stop` fail with the corrupt-session error and leaves Docker untouched; a hand-edited `progress.json` postmortem path pointing at a file outside `postmortems/` is not served by `breaklab serve` and not published by `breaklab publish`; and the full `BREAKLAB_INTEGRATION=1` suite still passes for all three `web-down` variants.

## Workflow follow-up

- Run `gofmt -l` / `go vet` and the full `go test ./...` suite before archiving.
- Run `openspec validate add-security-audit --strict` and fix any findings.
- Once the repo is pushed to GitHub, confirm `ci.yml`'s `govulncheck` job is green on the first run (the build plan's done-when for this change).
- Archive the change with `/opsx:archive` once the above pass and the user has reviewed it.
