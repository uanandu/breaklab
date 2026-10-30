# Tasks

## 1. Session and runner changes

- [x] 1.1 Add `HintsUsed int` to `Session` in `internal/runner/session.go`; verify with a unit test round-tripping a session with a non-zero `HintsUsed` through `SaveSession`/`LoadSession`.
- [x] 1.2 Implement `Runner.Hint(ctx) (string, error)` in `internal/runner/runner.go` (or a new `internal/runner/hint.go`): no active lab returns `ErrNoActiveLab`; a missing variant (per design.md Decision 6) returns a clear error; otherwise it concatenates the lab's hints with the active variant's hints, returns the one at index `HintsUsed`, increments and saves it; running out returns a new `ErrOutOfHints` naming `breaklab reset`. Verify with unit tests covering: first call returns the first general hint, calls progress through general then variant hints in order, the call after the last hint returns `ErrOutOfHints`, no active lab returns `ErrNoActiveLab`, and a session whose variant no longer exists in `lab.yaml` errors clearly.
- [x] 1.3 Zero `HintsUsed` when `Runner.Reset` writes its session (design.md Decision 4); verify with a unit test asserting hints used before a reset do not carry over, and a follow-up `Hint` call after reset returns the lab's first hint again.

## 2. CLI command

- [x] 2.1 Add `breaklab hint` to `cmd/breaklab/` (new file or alongside the other lab commands) and register it in `commands`/`commandOrder` in `cmd/breaklab/main.go`; on `ErrOutOfHints` print the message and exit non-zero, otherwise print the hint and exit 0. Verify with a CLI-level test asserting `breaklab hint --help` exits 0 and is listed in `commandOrder`.
- [x] 2.2 Verify by hand against a real lab: `breaklab start web-down`, then `breaklab hint` repeatedly until hints run out, confirming the message and the suggestion to `breaklab reset`, then confirm `breaklab reset` followed by `breaklab hint` returns the first hint again.

## Workflow follow-up

- Run `gofmt -l` / `go vet` and the full `go test ./...` suite before archiving.
- Run `openspec validate add-tiered-hints --strict` and fix any findings.
- Archive the change with `/opsx:archive` once the above pass and the user has reviewed it.
