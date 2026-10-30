## Why

BreakLab needs a way to discover and validate labs before it can run any
of them. This is change 1 of 9 in the v1 build plan: it lays down the Go
project skeleton and the lab catalog loader that every later change
(runner, variants, hints, progress, web UI) will read from, without yet
touching Docker, checks, or the web UI.

## What Changes

- New Go module (`go.mod`) with a `breaklab` CLI entry point at
  `cmd/breaklab/main.go`.
- `internal/catalog` package that loads and validates every
  `labs/<id>/lab.yaml` plus the root `roadmap.yaml` at startup, rejecting
  labs whose id, stage, or script paths are invalid, per the Lab catalog
  requirement word-for-word as written in `docs/breaklab-spec.md`.
- `breaklab list` command that prints labs grouped by roadmap stage, in
  roadmap order, each row showing id, title, difficulty, and status.
- `roadmap.yaml` at the repo root with the seven stages from the build
  spec (linux, git, docker, cicd, iac, kubernetes, observability).
- The `web-down` lab's data files: `labs/web-down/lab.yaml`,
  `labs/web-down/TICKET.md`, and `labs/web-down/Dockerfile`, matching the
  "Starter labs" section of the build spec. Variant and solution scripts
  are explicitly out of scope — they belong to change 3
  (`add-random-variants`) — but `lab.yaml` still declares the `variants:`
  entries (id, weight, hints) the catalog schema requires, even though
  their scripts don't exist on disk yet.
- `ruff`-equivalent tooling for Go: `gofmt` and `go vet` wired up as the
  lint/format check, plus `pytest`-equivalent `go test` for unit tests of
  the catalog loader.
- No Docker wrapper, no `start`/`check`/`reset`/`stop`, no variant
  selection, no hints, no progress tracking, no web UI, and no security
  audit — those are changes 2 through 9.

## Capabilities

### New Capabilities
- `lab-runner`: this change implements only the **Lab catalog**
  requirement (loading `lab.yaml`/`roadmap.yaml`, validating ids/stages,
  and `breaklab list`). The same spec file gains the remaining lab-runner
  requirements (starting, checking, variant selection, hints, reset/stop)
  in changes 2–4; this proposal's spec delta is scoped to Lab catalog only.

### Modified Capabilities
_None — this is the first change, nothing exists yet to modify._

## Impact

- Affected code: new `cmd/breaklab/`, `internal/catalog/`, `go.mod`,
  `go.sum`.
- Affected data: new `roadmap.yaml`, new `labs/web-down/` (lab.yaml,
  TICKET.md, Dockerfile only).
- No runtime dependencies beyond the Go standard library and
  `gopkg.in/yaml.v3`.
- No impact on Docker, the public site, or the local web UI — none of
  those exist yet.
