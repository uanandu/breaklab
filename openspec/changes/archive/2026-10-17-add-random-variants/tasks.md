# Tasks

## 1. Variant selection package

- [x] 1.1 Create `internal/variants` with a `Pick(variants []catalog.Variant, previous string, seed *int64) (catalog.Variant, error)` function implementing the cumulative-weight walk from design.md Decision 1; verify with a unit test that calls it 10,000 times over a 3-variant, uneven-weight fixture and asserts each variant's observed share is within a reasonable tolerance of `weight / total`.
- [x] 1.2 Implement the no-repeat exclusion (previous variant removed from the candidate list before weighting, skipped when `previous == ""` or the lab has one variant) with the weight-0-fallback from Decision 3; verify with a unit test asserting the previous variant is never returned when 2+ variants exist, across 1,000 seeded draws per variant.
- [x] 1.3 Implement `--seed` support (deterministic PCG-seeded draw over the full variant list, bypassing exclusion per Decision 4) and validate negative/zero weights are rejected with a clear error; verify with a unit test asserting two calls with the same seed and variant set return the same variant, and a test asserting a negative weight errors.
- [x] 1.4 Implement a small `.breaklab/history.json` reader/writer (`LastVariant(path, labID) (string, error)`, `RecordVariant(path, labID, variantID string) error`) matching `session.json`'s atomic-write style; verify with a unit test for round-trip read/write, a missing file returning "" with no error, and a corrupt file returning a named error.

## 2. Runner integration

- [x] 2.1 Replace `pickVariant` in `internal/runner/runner.go` with a call into `internal/variants`, wiring in `BREAKLAB_FORCE_VARIANT` (validated against `lab.Variants` by id per Decision 5, bypassing both exclusion and weighting), the lab id, and the history file path; verify with a `runner` unit test (fake Docker) asserting `BREAKLAB_FORCE_VARIANT` selects that exact variant and an unknown value errors without touching Docker.
- [x] 2.2 Add a `Seed *int64` field threaded from `Start` into selection, and record the selected variant to `history.json` only on a successful `Start` launch (never on `Reset`); verify with a unit test asserting `Reset` does not alter `history.json` and a successful `Start` does.
- [x] 2.3 Add the `--seed` flag to `breaklab start` in `cmd/breaklab/lab.go`, parsed as an int64; verify with a CLI-level test (or `go run` smoke check) that `breaklab start web-down --seed 42` run twice against a clean `.breaklab/` picks the same variant both times.

## 3. web-down variant and solution scripts

- [x] 3.1 Write `labs/web-down/variants/port-taken.sh` (binds a process to port 80 before nginx starts) and `labs/web-down/solutions/port-taken.sh` (frees the port so nginx can bind it), both `set -euo pipefail`; verify by running `breaklab start web-down` with `BREAKLAB_FORCE_VARIANT=port-taken` and confirming `breaklab check` fails, then running the solution script inside the container and confirming `breaklab check` passes.
- [x] 3.2 Write `labs/web-down/variants/bad-perms.sh` (removes read access to the homepage file for the user nginx's worker runs as) and `labs/web-down/solutions/bad-perms.sh` (restores it); verify the same way as 3.1 with `BREAKLAB_FORCE_VARIANT=bad-perms`.

## 4. Lab integration test coverage

- [x] 4.1 Extend the existing lab integration test (`internal/runner/integration_test.go`) to loop over all three web-down variants (`config-typo`, `port-taken`, `bad-perms`) using `BREAKLAB_FORCE_VARIANT`, asserting fail-then-pass-after-solution for each; verify by running `go test ./internal/runner/... -run Integration` locally with Docker available.
- [x] 4.2 Add a catalog-level or variants-level test asserting `web-down`'s `lab.yaml` variant entries all resolve to existing scripts on disk (closing the gap left by change 1, which allowed declaring variants without scripts); verify the test fails if a variant script is deleted and passes as shipped.

## Workflow follow-up

- Run `gofmt -l` / `go vet` and the full `go test ./...` suite (including the extended integration test) before archiving.
- Run `openspec validate add-random-variants --strict` and fix any findings.
- Try `breaklab start web-down` by hand a few times and confirm variants vary and never repeat back to back.
- Archive the change with `/opsx:archive` once the above pass and the user has reviewed it.
