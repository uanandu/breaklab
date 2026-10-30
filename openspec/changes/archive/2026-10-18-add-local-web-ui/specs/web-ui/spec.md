# Spec Delta

## Purpose

Unlike the public site, this is the view a learner opens every day: it
reads progress.json, roadmap.yaml, lab metadata, and their postmortems
live, with no publish step, answering "I want a roadmap/challenges/
dashboard like devopslabs.dev, but local and mine."

## ADDED Requirements

### Requirement: Local server
The system SHALL serve a local web UI with `breaklab serve [--port] [--host]`,
bound to 127.0.0.1 by default, and SHALL open the user's default browser
to it when bound to a loopback address.

#### Scenario: Starting the server
- **WHEN** the user runs `breaklab serve`
- **THEN** the CLI starts a local HTTP server on 127.0.0.1 (default port 8787)
- **AND** prints the URL
- **AND** opens the URL in the default browser

#### Scenario: Starting on a headless host
- **WHEN** the user runs `breaklab serve --host 0.0.0.0` (or any other
  non-loopback address, as on a headless Raspberry Pi)
- **THEN** the CLI starts the server bound to that address and prints the URL
- **AND** does NOT attempt to open a local browser

#### Scenario: Port already in use
- **WHEN** the chosen port is already bound
- **THEN** the CLI prints a clear error naming the port and exits non-zero,
  rather than silently picking a different one

### Requirement: Dashboard view
The system SHALL render a dashboard showing the current streak, totals
(labs completed, variants seen), the active lab with elapsed time and
hints used if one is running, and the most recent postmortems.

#### Scenario: Dashboard with an active lab
- **WHEN** a lab is active and the user opens the dashboard
- **THEN** it shows the lab's elapsed time, hints used, and a reminder of the
  command to check it

#### Scenario: Dashboard with no active lab
- **WHEN** no lab is active
- **THEN** the dashboard shows totals and streak only, with a prompt to start
  a lab from the roadmap or challenge browser

### Requirement: Roadmap view
The system SHALL render every stage from roadmap.yaml in order, and every
lab under its stage, each labeled with status (available, in-progress,
complete), difficulty, and estimated minutes.

#### Scenario: Viewing the roadmap
- **WHEN** the user opens the roadmap page
- **THEN** stages appear in roadmap.yaml order
- **AND** each lab under a stage shows status, difficulty, and estimated minutes
- **AND** a completed lab links to its postmortem if one exists

### Requirement: Challenge browser view
The system SHALL render a browsable list of all labs across every stage,
filterable by stage, difficulty, and status.

#### Scenario: Filtering challenges
- **WHEN** the user filters the challenge browser by stage and difficulty
- **THEN** only matching labs are shown
- **AND** clearing the filters restores the full list

### Requirement: Lab detail view
The system SHALL render a lab's ticket text, status, hints used so far,
and the exact CLI command to start or resume it, for any lab the user
selects from the roadmap or challenge browser.

#### Scenario: Opening a lab's detail page
- **WHEN** the user opens a lab from the roadmap or challenge browser
- **THEN** the ticket text, status, and hints-used count are shown
- **AND** the CLI command to start or resume it is shown
- **AND** no variant id, unused hint text, or any variants/ or solutions/
  content ever appears, whether or not the lab is solved

### Requirement: Postmortem viewer
The system SHALL render any of the user's own completed postmortems in
the local web UI regardless of their publish flag, since that flag only
controls what becomes public.

#### Scenario: Viewing a local postmortem
- **WHEN** the user opens a completed lab's postmortem from the local UI
- **THEN** its full content renders, even when publish: false

### Requirement: Configurable bind host, still unauthenticated
The system SHALL bind to 127.0.0.1 unless the user explicitly passes
`--host`, and SHALL NOT require or implement authentication regardless
of which host it binds to.

#### Scenario: Default binding stays local
- **WHEN** the user runs `breaklab serve` with no `--host` flag
- **THEN** the server binds to 127.0.0.1 only, matching the non-goal of no
  authentication for the one person running it

#### Scenario: LAN binding is an explicit opt-in
- **WHEN** the user runs `breaklab serve --host 0.0.0.0` (or a specific
  LAN address), as when deploying to a headless Raspberry Pi
- **THEN** the server binds to that address
- **AND** the CLI prints a one-time warning that the UI is unauthenticated
  and reachable by any device that can reach that address
