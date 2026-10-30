## MODIFIED Requirements

### Requirement: CI supply-chain hardening
GitHub Actions workflows SHALL pin third-party actions to a full commit
SHA rather than a mutable tag, and SHALL NOT use `pull_request_target`
with untrusted checkout. An automated dependency-update mechanism SHALL
keep those pins and the Go toolchain version current, so staleness is
caught by routine update proposals rather than by a point-in-time audit.

#### Scenario: Reviewing workflow files
- **WHEN** the audit checks ci.yml and publish.yml
- **THEN** every `uses:` line for a non-GitHub-owned action references a
  commit SHA, not a tag like `@v4`

#### Scenario: Pinned action goes stale
- **WHEN** a pinned GitHub Action has a newer release than the commit SHA
  currently pinned in a workflow file
- **THEN** an automated dependency update proposes the new SHA, resolved
  against the action's real upstream release, rather than the pin going
  unnoticed until the next manual audit

#### Scenario: Go toolchain goes stale
- **WHEN** a newer Go patch release exists than the `go` directive pinned
  in go.mod
- **THEN** an automated dependency update proposes the bump, so a
  `govulncheck` finding introduced only by stdlib drift surfaces as a
  routine update rather than an unexplained CI failure on an unrelated PR
