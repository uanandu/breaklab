## MODIFIED Requirements

### Requirement: Dashboard view
The system SHALL render a dashboard showing the current streak, totals
(labs completed, variants seen), the active lab with elapsed time and
hints used if one is running, and the most recent postmortems. When the
request carries a valid hosted-account session, the dashboard SHALL
read/merge attempts from that account's hosted store in addition to the
local progress.json; with no session, it SHALL behave exactly as before,
reading only the local file.

#### Scenario: Dashboard with an active lab
- **WHEN** a lab is active and the user opens the dashboard
- **THEN** it shows the lab's elapsed time, hints used, and a reminder of the
  command to check it

#### Scenario: Dashboard with no active lab
- **WHEN** no lab is active
- **THEN** the dashboard shows totals and streak only, with a prompt to start
  a lab from the roadmap or challenge browser

#### Scenario: Dashboard with no account session
- **WHEN** the request carries no hosted-account session
- **THEN** the dashboard reads only the local progress.json, identical
  to its behavior before accounts existed

#### Scenario: Dashboard with an account session
- **WHEN** the request carries a valid hosted-account session
- **THEN** the dashboard's totals and streak reflect attempts synced to
  that account, merged with any local-only attempts not yet synced

### Requirement: Roadmap view
The system SHALL render every stage from roadmap.yaml in order, and every
lab under its stage, each labeled with status (available, in-progress,
complete), difficulty, and estimated minutes. When the request carries
a valid hosted-account session, lab status SHALL be computed from that
account's merged (local + synced) attempts; with no session, it SHALL
behave exactly as before, using only the local progress.json.

#### Scenario: Viewing the roadmap
- **WHEN** the user opens the roadmap page
- **THEN** stages appear in roadmap.yaml order
- **AND** each lab under a stage shows status, difficulty, and estimated minutes
- **AND** a completed lab links to its postmortem if one exists

#### Scenario: Roadmap with no account session
- **WHEN** the request carries no hosted-account session
- **THEN** lab status is computed only from the local progress.json,
  identical to its behavior before accounts existed

#### Scenario: Roadmap reflects progress synced from another device
- **WHEN** the request carries a valid hosted-account session, and an
  attempt for a given lab was synced from a different machine
- **THEN** that lab's status reflects the synced attempt even though it
  was never run on this machine
