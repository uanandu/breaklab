## 1. Toolchain freshness (already verified this session, formalize here)

- [x] 1.1 Confirm `go.mod`'s `go` directive is the latest `1.26.x` patch (bumped from `1.26.1` to `1.26.9` during this change's QA pass). Verify `go run golang.org/x/vuln/cmd/govulncheck@v1.9.0 ./...` reports no vulnerabilities, and `gofmt -l .`, `go vet ./...`, and `go test ./...` all still pass.

## 2. Dependabot configuration

- [x] 2.1 Add `.github/dependabot.yml` with a `version: 2` top level and two `updates` entries: `package-ecosystem: "github-actions"` watching `directory: "/"` on a weekly schedule, and `package-ecosystem: "gomod"` watching `directory: "/"` on a weekly schedule. Verify the file parses as YAML.
- [x] 2.2 Add a regression test in `internal/audit` (e.g. `dependabot_test.go`) asserting `.github/dependabot.yml` exists, is valid YAML, and its `updates` list contains at least one entry with each of `package-ecosystem: github-actions` and `package-ecosystem: gomod`. Verify the test fails if the file is deleted or either ecosystem entry is removed (check by temporarily editing a scratch copy, not the real file), then passes against the real file.

## 3. Cleanup (non-spec, out-of-band per proposal.md)

- [x] 3.1 Remove the Python-tooling entries (`.venv/`, `__pycache__/`, `*.pyc`, `.pytest_cache/`, `.ruff_cache/`) from `.gitignore`. Verify `git status` shows no newly-untracked files as a result (none of those paths exist in this repo) and the existing `internal/audit` gitignore-coverage test still passes.

## 4. Verification

- [x] 4.1 Run `gofmt -l .`, `go vet ./...`, and `go test ./...`; all must pass with no new failures.
- [x] 4.2 Run `openspec validate add-dependency-automation --strict` and fix any findings.

## Workflow follow-up

- Sync the `security` capability's modified requirement into the main spec with `/opsx:sync` once implementation is complete.
- Archive with `/opsx:archive` once tasks and sync are done and the user has reviewed it.
- Once the repo has a GitHub remote, confirm Dependabot actually opens its first PR (not a blocking check for archiving this change — Dependabot only runs once the repo is pushed).
