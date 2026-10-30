# lab-runner Specification

## Purpose
Lets BreakLab discover, validate, and list the labs on disk so every
later command (start, check, hint, reset, stop) has a trustworthy
catalog of labs and roadmap stages to work from.

## Requirements

### Requirement: Lab catalog
The system SHALL load every `labs/<id>/lab.yaml` and `roadmap.yaml` at
startup and reject labs whose id, stage, script paths, or tags are invalid.

#### Scenario: Listing labs
- **WHEN** the user runs `breaklab list`
- **THEN** labs are shown grouped by roadmap stage, in roadmap order
- **AND** each row shows id, title, difficulty, and status

#### Scenario: Invalid lab
- **WHEN** a lab.yaml references a stage missing from roadmap.yaml
- **THEN** `breaklab list` prints a warning naming the lab and the problem
- **AND** the other labs still load

### Requirement: Starting a lab
The system SHALL build the lab image, start a fresh container, and apply
exactly one variant chosen by the variant selector.

#### Scenario: Clean start
- **WHEN** the user runs `breaklab start web-down`
- **THEN** any previous container for that lab is removed first
- **AND** a container named breaklab-web-down is started with the lab's ports
- **AND** the selected variant script runs inside it as root
- **AND** the ticket text and the `docker exec` command to get a shell are printed
- **AND** the variant id is NOT printed

#### Scenario: Another lab already active
- **WHEN** a different lab is active and the user starts a new one
- **THEN** the CLI asks for confirmation before stopping the active lab

#### Scenario: Docker not running
- **WHEN** the docker CLI is missing or the daemon is unreachable
- **THEN** the CLI exits with code 2 and a message explaining how to start Docker

#### Scenario: A lab opts into a privileged container
- **WHEN** a lab's `environment.privileged` is `true` (for example, a
  Kubernetes lab whose control plane needs cgroup access an unprivileged
  container is denied)
- **THEN** its container is started with `--privileged`
- **AND** every lab that does not set `environment.privileged: true`
  starts as an ordinary, unprivileged container

### Requirement: Random variant selection
The system SHALL pick a variant at random, weighted by `weight`, and SHALL
not pick the variant used on that lab's previous attempt when the lab has
two or more variants.

#### Scenario: No immediate repeat
- **WHEN** web-down's last attempt used config-typo
- **AND** the user starts web-down again
- **THEN** the selected variant is not config-typo

#### Scenario: Reproducible selection
- **WHEN** the user runs `breaklab start web-down --seed 42` twice
- **THEN** the same variant is selected both times

#### Scenario: Forced variant for tests
- **WHEN** the BREAKLAB_FORCE_VARIANT environment variable is set
- **THEN** that variant is used and the selection rules are skipped

### Requirement: Checking a fix
The system SHALL run the lab's check script and report pass or fail with
the script's one-line message.

#### Scenario: Fix works
- **WHEN** `breaklab check` runs and the check script exits 0
- **THEN** the CLI prints the message and reveals the variant id

#### Scenario: Not fixed yet
- **WHEN** the check script exits non-zero
- **THEN** the CLI prints the message

#### Scenario: No active lab
- **WHEN** the user runs `breaklab check` and no lab is active
- **THEN** the CLI exits non-zero with a message to run `breaklab start <lab>`

### Requirement: Tiered hints
The system SHALL reveal one hint per `breaklab hint` call: general hints
first, then the active variant's hints, and count how many were used.

#### Scenario: Out of hints
- **WHEN** every hint has been shown
- **THEN** the CLI says so and suggests `breaklab reset`

### Requirement: Reset and stop
The system SHALL let the user retry with `breaklab reset` (same variant,
fresh container) and quit with `breaklab stop` (container removed).

#### Scenario: Reset keeps the variant
- **WHEN** the user runs `breaklab reset` with a lab active
- **THEN** the container is removed and recreated from a fresh image build
- **AND** the same variant script is applied again
- **AND** the variant id is NOT printed

#### Scenario: Stop cleans up
- **WHEN** the user runs `breaklab stop` with a lab active
- **THEN** the container is removed
- **AND** no lab is active afterwards
