## 1. Lab data for web-down

- [x] 1.1 Create `labs/web-down/check.sh` exactly as given in `docs/breaklab-spec.md`'s "Starter labs" section; verify `bash -n` passes and the file is executable
- [x] 1.2 Create `labs/web-down/variants/config-typo.sh` and `labs/web-down/solutions/config-typo.sh` (a reference fix that repairs it); verify `bash -n` passes on both and `shellcheck` is clean if installed

## 2. Docker wrapper (internal/docker)

- [x] 2.1 Define the `Docker` type with an injectable command runner, `ErrUnavailable`, and `Ping` (`docker info`); verify unit tests with a fake runner cover missing binary, non-zero exit, and success
- [x] 2.2 Implement `Build`, `Run` (name, image, `-p` mappings), `Exec`, `Copy`, `Remove` (idempotent when absent), and `Exists`, validating names against `^[a-z0-9-]+$` and building every command as an argument slice; verify unit tests assert the exact argv and that an id like `a;b` is rejected
- [x] 2.3 Verify by test that no code path in `internal/docker` or `internal/runner` invokes `sh -c` or `bash -c`

## 3. Session state (internal/runner)

- [x] 3.1 Implement load/save/clear of `.breaklab/session.json` (lab, container, variant, started_at) with atomic write and mode 0600; verify tests for missing file (no active lab), round-trip, and corrupt file (error names the path)
- [x] 3.2 Implement `pickVariant(lab)` returning the first variant and an error for a lab with none; verify with a unit test

## 4. Runner operations

- [x] 4.1 Implement `Start`: ping Docker, handle an active different lab, remove the old container, build, run with ports, copy and run the variant script as root, remove the script, write the session; verify a fake-Docker unit test asserts the call order and that the variant id appears in no returned output
- [x] 4.2 Implement the confirmation flow (`[y/N]`, non-interactive = no, `--yes` bypass); verify unit tests for yes, no, and non-interactive
- [x] 4.3 Implement port parsing into `BREAKLAB_PORT_<container>` env entries and reject malformed ports; verify `8080:80` gives `BREAKLAB_PORT_80=8080`
- [x] 4.4 Implement `Check` for `run_in: host` and `run_in: container` with `BREAKLAB_CONTAINER`, a 30-second timeout, and a clear error for a missing script; verify tests for pass, fail, missing script, and no active lab
- [x] 4.5 Implement `Reset` (same variant, rebuild, recreate) and `Stop` (remove container, clear session); verify tests that reset keeps the variant and stop clears the session

## 5. CLI commands

- [x] 5.1 Wire `start <lab> [--yes]`, `check`, `reset`, `stop` into the dispatch map in `cmd/breaklab` with per-command `--help`; verify `breaklab start --help` prints usage and an unknown lab id exits non-zero with a message
- [x] 5.2 Map `docker.ErrUnavailable` to exit code 2 with the start-Docker message; verify with a test using a fake that returns it
- [x] 5.3 `start` prints the ticket and the `docker exec -it breaklab-<lab> bash` command and never the variant id; `check` reveals the variant id only on a pass; verify output tests for both

## 6. Integration, lint, and manual check

- [x] 6.1 Add an integration test, skipped unless `BREAKLAB_INTEGRATION=1`, that starts web-down, confirms `check.sh` fails, applies the reference solution via `docker exec`, and confirms `check.sh` passes
- [x] 6.2 Run `gofmt -l .`, `go vet ./...`, and `go test ./...`; verify no problems
- [x] 6.3 Manually run `breaklab start web-down`, break-and-fix by hand, `breaklab check`, `breaklab reset`, `breaklab stop`; confirm `docker ps -a` shows no `breaklab-` containers afterwards
