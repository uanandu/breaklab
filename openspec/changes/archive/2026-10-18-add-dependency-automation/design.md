## Context

See proposal.md - Why for the two gaps this closes. Both were found by
actually running the system, not by inspection: a hand-run `govulncheck`
against the `go.mod`-pinned toolchain, and a read of `internal/audit`'s
existing SHA-pin test (`internal/audit/workflows_test.go`), which already
passes today — the gap is that nothing would catch a *future* stale or
incorrect pin, since the test only checks SHA shape (40 hex chars), never
freshness or correctness against the real upstream tag.

## Goals / Non-Goals

**Goals:**
- Give both pinned GitHub Actions and the Go toolchain version an
  automated path to staying current.
- Make the "automated update mechanism exists and covers both" guarantee
  itself a regression test, consistent with how `add-security-audit`
  turned every other audit finding into a test.

**Non-Goals:**
- Auto-merging dependency update PRs. A one-person project reviews and
  merges them by hand; this change only makes the proposals appear.
- Re-verifying SHA-correctness inside `go test` via a live `git
  ls-remote` call. Unit tests shouldn't depend on network reachability;
  Dependabot's own resolution against GitHub's API is the correctness
  check, running on its own schedule instead of on every `go test` run.
- Touching the `.gitignore` Python-tooling entries' underlying cause (how
  they got there) — just removing the dead weight.

## Decisions

**1. Dependabot over a custom `ls-remote` verifier.**
A hand-rolled test that shells out to `git ls-remote` on every `go test`
run would make the test suite flaky offline and slow in CI, and would
only catch a bad pin, not propose the fix. `github-actions` Dependabot
natively parses SHA-pinned `uses:` lines (including the `# vN` trailing
comment convention already used in `ci.yml`/`publish.yml`) and opens a PR
with the correct new SHA when a release ships. This is strictly less
code than a custom checker and is verified against GitHub's own data,
not a local clone's view of tags.

**2. `gomod` ecosystem for the Go toolchain, not a custom version check.**
Dependabot's `gomod` ecosystem reads the `go` directive in `go.mod` and
proposes bumps, including ones driven by the OSV/GHSA advisories
`govulncheck` itself consults. This closes the exact gap that just
bit this session (a stale toolchain making `govulncheck` fail on
unrelated code) with the same "routine PR instead of a surprise" shape
as decision 1.

**3. One `dependabot.yml`, weekly interval, both ecosystems, grouped by
default Dependabot behavior (no custom grouping needed for two
ecosystems on one module).** A monthly interval would leave the toolchain
stale for up to a month after a security patch ships; daily is more
noise than a one-person project benefits from. Weekly matches this
project's existing audit cadence (manual review before archiving each
change).

**4. The regression test checks for the file and both `package-ecosystem`
values as a string/structural check, not a full schema validation of
Dependabot's config grammar.** `internal/audit` already does a syntactic
check style (its workflow test regex-matches `uses:` lines); matching
that style keeps the new test consistent and avoids embedding a YAML
schema for a file only this project's CI interprets.

## Risks / Trade-offs

- [Dependabot PRs require review before merge, so a lapse in manual
  review re-creates the staleness this change tries to prevent] →
  Mitigation: this is an explicit, accepted trade-off for a one-person
  project (see Non-Goals); the test only guarantees the *mechanism*
  exists, not that every PR gets merged promptly.
- [Dependabot is a GitHub-hosted feature; it does nothing until the repo
  is actually pushed to GitHub] → Mitigation: already true of this
  project's state (no remote configured yet) and out of scope for this
  change; `dependabot.yml` is inert but harmless until then, and the
  regression test only checks the file's presence and contents, not that
  Dependabot has run.

## Migration Plan

Additive only: one new file (`.github/dependabot.yml`), one `go.mod`
line already bumped, one `.gitignore` trim, one new test file. No
rollback concerns beyond `git revert`.
