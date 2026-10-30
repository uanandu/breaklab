## ADDED Requirements

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
