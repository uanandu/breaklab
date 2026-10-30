# Proposal

## Why

BreakLab shells out to Docker, runs lab scripts as root inside
containers, serves a web UI that can be LAN-reachable, and publishes a
public site built from the learner's own files. This is change 9 of 9 in
the v1 build plan: a deliberate hardening pass over what changes 1–8
built, rather than an assumption that "it's just for me" makes it safe.
A survey of the code before writing this proposal found that most of the
audit already holds (no `sh -c` anywhere, every `exec.Command` call uses
an argument slice, catalog ids are regex-validated, web routes resolve
against allowlists), but also found three concrete gaps and two missing
CI pieces, listed below.

## What Changes

- **Fix: session fields reach Docker unvalidated.** `.breaklab/session.json`'s
  `container` value is passed straight to `docker rm`/`exec`/`cp`. The
  Docker wrapper's `^[a-z0-9-]+$` check stops path and shell characters
  but still accepts a value like `-f` or `--help`, which docker would read
  as a flag. `LoadSession` will reject a session whose `lab` fails the id
  pattern or whose `container` isn't exactly `breaklab-<lab>`.
- **Fix: postmortem filenames are built from unvalidated `progress.json`
  fields.** `breaklab postmortem` builds `postmortems/<date>-<lab>-<variant>.md`
  from an attempt's `lab`/`variant`. `progress.json` is a hand-editable,
  committed file, so an edited `variant` like `../../x` would escape
  `postmortems/`. Both ids will be validated against `^[a-z0-9-]+$` first.
- **Fix: postmortem *reads* trust any path recorded in `progress.json`.**
  `breaklab publish`, `breaklab postmortem --validate`, and the web UI's
  postmortem viewer all read whatever file an attempt's `postmortem`
  field names. On a LAN-exposed `breaklab serve`, an edited path pointing
  outside `postmortems/` would be read and served. Reads will be
  confined to files directly inside the postmortems directory.
- **Add `.github/workflows/ci.yml`.** No earlier change created it,
  though `docs/breaklab-spec.md`'s "Testing and definition of done"
  describes it: `gofmt`/`go vet`, `go test ./...`, the lab integration
  test, a `linux/arm64` cross-compile smoke check, and, from this change,
  a `govulncheck` step that fails the build on findings.
- **Pin every GitHub Action to a full commit SHA** in `ci.yml` and
  `publish.yml`, resolved from the real upstream tags with
  `git ls-remote`, never written from memory (see design.md).
- **Confirmed, no code change needed, and locked in with tests:** no
  shell execution anywhere; `breaklab serve` binds `127.0.0.1` unless
  `--host` is passed; web routes accept no filesystem path or command
  from requests; `.gitignore` covers `.breaklab/` and `site/`; the public
  site's "nothing leaks" and "works offline" guarantees still hold.
- Out of scope: new user-facing features, authentication for the web UI
  (an explicit v1 non-goal), and anti-cheating.

## Capabilities

### New Capabilities

- `security`: safe process execution, input/path validation, no secret
  leakage, a CI dependency audit, CI supply-chain hardening, and a
  bounded web UI attack surface. These are the cross-cutting guarantees
  the other five capabilities rely on.

### Modified Capabilities

_None._ The three fixes tighten how existing behavior handles malformed
or tampered input files. They don't change what `lab-runner`,
`postmortems`, `progress-site`, or `web-ui` specify for valid input, so
none of their requirement text changes.

## Impact

- Affected code: `internal/runner/session.go` (`LoadSession` validation);
  `internal/runner/postmortem.go` (id validation before building a path,
  confined reads); `internal/site/postmortem.go` and
  `internal/web/postmortem.go` (confined reads); a small shared
  id/path-confinement helper; new audit tests.
- Affected CI: new `.github/workflows/ci.yml`; `.github/workflows/publish.yml`
  (SHA-pinned actions). No other behavior changes.
- No new Go dependencies. `govulncheck` runs in CI via `go run
  golang.org/x/vuln/cmd/govulncheck@<pinned version>` and isn't added to `go.mod`.
- Operational note: running `govulncheck` locally from this sandbox
  needs `vuln.go.dev` approved in the sandbox network policy (an approval
  is already pending).
