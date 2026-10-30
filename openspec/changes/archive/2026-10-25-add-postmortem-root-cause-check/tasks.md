## 1. Word-overlap check

- [x] 1.1 Add a small fixed stopword list and a word-tokenize helper (lowercase, strip punctuation, drop words under 4 characters) to `internal/postmortem`, verified with a unit test on sample sentences
- [x] 1.2 Add `rootCauseMentionsVariant(rootCauseText, variantHints []string) bool` checking for any token overlap, verified with unit tests covering: overlap found, no overlap, empty hints

## 2. Wire into validate

- [x] 2.1 Resolve the solved attempt's actual variant from the catalog during `--validate` and pass its `Hints` to the new check, verified with a test using a fixture lab/variant
- [x] 2.2 Print a warning line (naming the expected words) when the check finds no overlap, without affecting exit code, verified with a CLI integration test asserting exit code 0 and the warning text both appear — exercised at the `internal/runner` level (this codebase's existing CLI-behavior-testing layer; `cmd/breaklab` itself has no deeper tests than flag parsing for any command) via `TestValidatePostmortemRootCauseMismatchWarnsButStillCompletes`/`TestValidatePostmortemRootCauseMatchHasNoWarning`, plus the one-line print wired into `cmd/breaklab/postmortem.go`

## 3. End-to-end verification

- [x] 3.1 Run the full test suite (`go test ./...`) and confirm no regression in existing postmortem validation tests
- [x] 3.2 Manually validate one real postmortem with a correct root cause (no warning) and one edited to describe an unrelated cause (warning appears, still completes) — covered by equivalent automated tests instead of a live walkthrough, per explicit user confirmation
