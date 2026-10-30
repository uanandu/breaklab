## 1. Project setup

- [x] 1.1 Initialize the Go module (`go mod init`) and add `gopkg.in/yaml.v3` as a dependency; verify `go build ./...` succeeds with no source files beyond a placeholder `main.go`
- [x] 1.2 Create `cmd/breaklab/main.go` with a stdlib `flag`-based subcommand dispatcher that recognizes `list` and prints a clear "unknown command" error plus usage for anything else; verify `go run ./cmd/breaklab bogus` exits non-zero with a usage message and `go run ./cmd/breaklab --help` prints help text
- [x] 1.3 Add `.gitignore` entries for `.breaklab/` and `site/`; verify both patterns are present

## 2. Roadmap and lab data

- [x] 2.1 Create `roadmap.yaml` at the repo root with the seven stages from `docs/breaklab-spec.md` (linux, git, docker, cicd, iac, kubernetes, observability), each with `id` and `title`; verify the file parses as valid YAML
- [x] 2.2 Create `labs/web-down/lab.yaml` per the "Lab format" schema: id `web-down`, stage `linux`, difficulty, estimated_minutes, `environment.dockerfile`/`ports`, `check.script`/`check.run_in`, general hints, and the three `variants` entries (`config-typo`, `port-taken`, `bad-perms`) with weights and per-variant hints, pointing at `variants/<id>.sh` paths that don't exist on disk yet; verify the file parses as valid YAML and matches the schema field-for-field
- [x] 2.3 Create `labs/web-down/TICKET.md` with the INC-101 ticket text from `docs/breaklab-spec.md`'s "Starter labs" section; verify the file exists and contains no mention of a variant or cause
- [x] 2.4 Create `labs/web-down/Dockerfile` exactly as given in `docs/breaklab-spec.md`'s "Starter labs" section; verify `docker build` on it succeeds (build-only check — this change does not run the container)

## 3. Catalog loader (internal/catalog)

- [x] 3.1 Define Go structs for `roadmap.yaml` (stages) and `lab.yaml` (id, title, stage, difficulty, estimated_minutes, environment, check, hints, variants) matching the "Lab format" schema; verify they decode `roadmap.yaml` and `labs/web-down/lab.yaml` without error using strict/`KnownFields` decoding
- [x] 3.2 Implement id/stage validation against `^[a-z0-9-]+$` for lab id, stage id, and variant ids, and shape validation (non-empty, relative, no `..`, contained within the lab directory) for all script paths (`check.script`, each variant's `script`); verify unit tests cover a passing lab, a bad id, a bad stage reference, and an escaping script path
- [x] 3.3 Implement `LoadCatalog(labsDir, roadmapPath string)` that loads `roadmap.yaml`, walks `labs/*/lab.yaml`, validates each, and returns the valid labs grouped by roadmap stage plus a list of per-lab warnings (e.g. unknown stage) without aborting the whole load; verify a unit test with one valid lab and one lab referencing a missing stage returns the valid lab plus one warning, per the Invalid lab scenario
- [x] 3.4 Implement status computation for a lab as `available` when no progress data is present (no `progress/progress.json` reader exists yet in this change); verify a unit test asserts `web-down`'s computed status is `available`

## 4. CLI `list` command

- [x] 4.1 Wire `breaklab list` to call `LoadCatalog` and render labs grouped by roadmap stage, in roadmap order, each row showing id, title, difficulty, and status; verify running `go run ./cmd/breaklab list` shows `web-down` under the `Linux fundamentals` stage heading
- [x] 4.2 Print any per-lab warnings from `LoadCatalog` to stderr, naming the lab and the problem, without stopping the other labs from printing; verify a unit/integration test with a deliberately broken second lab still lists `web-down` and prints one warning line for the broken one

## 5. Tests, lint, and docs

- [x] 5.1 Run `gofmt -l .` and `go vet ./...` and fix any reported issues; verify both commands report no problems
- [x] 5.2 Run `go test ./...` and confirm all catalog and CLI tests pass
- [x] 5.3 Manually run `breaklab list` and confirm by hand that `web-down` appears under Linux fundamentals with the expected difficulty and `available` status
