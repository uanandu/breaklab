# Spec Delta

## Purpose

Makes BreakLab's security properties deliberate and checkable rather
than assumed: it shells out to Docker, runs lab scripts as root inside
containers, serves a web UI that can be LAN-reachable, and publishes a
public site, so process execution, input handling, secret exposure,
dependencies, CI, and the web UI's surface each get an explicit rule.

## ADDED Requirements

### Requirement: Safe process execution
The system SHALL invoke docker and lab scripts via `os/exec` using
argument slices, and SHALL NOT shell out through `sh -c` or build a
command by string interpolation anywhere in the codebase.

#### Scenario: Auditing exec calls
- **WHEN** the security audit runs
- **THEN** every `exec.Command` call under internal/ and cmd/ is checked
- **AND** any call that goes through a shell or a string-interpolated
  command is flagged and fixed before the audit passes

### Requirement: Input and path validation
The system SHALL validate lab ids, variant ids, and stage ids against
`^[a-z0-9-]+$` before using them to build a filesystem path, a Docker
container/image name, or a URL route, and SHALL reject anything else.

#### Scenario: Path traversal attempt
- **GIVEN** a lab.yaml, CLI argument, or URL path segment containing `..`,
  `/`, or other characters outside `^[a-z0-9-]+$`
- **WHEN** the system would use it to build a path or container name
- **THEN** the system rejects it with a clear error instead of touching the
  filesystem or Docker

### Requirement: No secret leakage
The system SHALL keep session state out of version control and SHALL
only expose postmortems through the public site when their front matter
says publish: true.

#### Scenario: Gitignore coverage
- **WHEN** the audit checks .gitignore
- **THEN** .breaklab/ and site/ are present
- **AND** no progress, postmortem, or session file is tracked that shouldn't be

#### Scenario: Unpublished content stays local
- **WHEN** `breaklab publish` runs
- **THEN** no file, filename, or text from a postmortem with publish: false,
  or from any variants/ or solutions/ script, appears anywhere in site/

### Requirement: Dependency audit
CI SHALL run `govulncheck` on every pull request and SHALL fail the
build on any finding that affects code reachable from main.

#### Scenario: Vulnerable dependency introduced
- **WHEN** a pull request adds or bumps a dependency with a known
  vulnerability reachable from the module's code
- **THEN** the CI audit step fails and blocks the merge

### Requirement: CI supply-chain hardening
GitHub Actions workflows SHALL pin third-party actions to a full commit
SHA rather than a mutable tag, and SHALL NOT use `pull_request_target`
with untrusted checkout.

#### Scenario: Reviewing workflow files
- **WHEN** the audit checks ci.yml and publish.yml
- **THEN** every `uses:` line for a non-GitHub-owned action references a
  commit SHA, not a tag like `@v4`

### Requirement: Web UI attack surface
The local web UI SHALL expose only read routes over the data it already
documents (roadmap, labs, progress, postmortems) and SHALL NOT include
any route that executes a shell command, reads an arbitrary path from a
query parameter, or writes to the filesystem. Binding to a non-loopback
host SHALL require the explicit `--host` flag from the web-ui spec and
SHALL NOT be the default.

#### Scenario: Reviewing web.go routes
- **WHEN** the audit checks every route in internal/web/web.go
- **THEN** none of them accept a filesystem path or shell command from the
  request and act on it directly

#### Scenario: LAN exposure requires opt-in
- **WHEN** the audit checks the default `breaklab serve` invocation
- **THEN** it binds to 127.0.0.1 unless `--host` was explicitly passed
