## Purpose

Moves a learner's `progress.Attempt` records between their local
`progress.json` and their hosted account, so progress follows them
across machines while every lab still runs only on the machine the
learner is sitting at.

## ADDED Requirements

### Requirement: CLI login links a local install to an account
The system SHALL provide `breaklab login --server <url> --token
<token>`, which validates the token against the given hosted server
and, on success, stores both locally for use by `breaklab sync`. Both
flags are required: there is no default server, since this project
ships as a self-hostable binary rather than pointing at one
fixed hosted instance.

#### Scenario: Valid token
- **WHEN** the user runs `breaklab login --server <url> --token <valid-token>`
- **THEN** the CLI confirms which account it linked to
- **AND** stores the server URL and token under `.breaklab/`
  (gitignored), not inside `progress/progress.json`

#### Scenario: Invalid or revoked token
- **WHEN** the user runs `breaklab login --server <url> --token <invalid-token>`
- **THEN** the CLI prints a clear error and stores nothing
- **AND** any previously stored valid server/token is left untouched

#### Scenario: Missing required flag
- **WHEN** the user runs `breaklab login` with `--server` or `--token`
  omitted
- **THEN** the CLI prints usage and exits non-zero without storing or
  validating anything

### Requirement: Sync pushes and pulls attempts
The system SHALL provide `breaklab sync`, which, when a token is
linked, pushes every local attempt not yet recorded on the hosted
account and pulls every hosted attempt not yet present locally, merging
by attempt identity rather than overwriting either side wholesale.

#### Scenario: First sync from a fresh machine
- **WHEN** the user runs `breaklab login` then `breaklab sync` on a
  machine with an empty `progress.json`
- **THEN** every attempt previously synced from any other machine for
  that account appears in the local `progress.json` afterward

#### Scenario: Pushing new local progress
- **WHEN** the user solves a lab locally and then runs `breaklab sync`
- **THEN** the hosted account gains that attempt
- **AND** running sync again does not duplicate it

#### Scenario: No token linked
- **WHEN** the user runs `breaklab sync` without having run
  `breaklab login`
- **THEN** the CLI prints an error telling them to log in first
- **AND** `progress.json` is left completely unchanged

#### Scenario: Sync never touches lab execution
- **WHEN** `breaklab sync` runs, with or without an active lab
- **THEN** no Docker container is started, stopped, or queried as part
  of the sync

### Requirement: Sync API is scoped to the authenticated account
The system SHALL expose `/sync/attempts` (GET and POST), authenticated
by bearer API token, and SHALL only ever read or write attempts
belonging to the account that token resolves to.

#### Scenario: Missing or invalid token
- **WHEN** a request to `/sync/attempts` carries no token or an
  invalid/revoked one
- **THEN** the system rejects it with 401 and returns no attempt data

#### Scenario: Cross-account isolation
- **WHEN** account A's token is used to call `/sync/attempts`
- **THEN** the response never includes any attempt belonging to a
  different account
