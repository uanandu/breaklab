# web-ui Specification

## Purpose
Unlike the public site, this is the view a learner opens every day: it
reads progress.json, roadmap.yaml, lab metadata, and their postmortems
live, with no publish step, answering "I want a roadmap/challenges/
dashboard like devopslabs.dev, but local and mine."

## Requirements

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

### Requirement: Challenge browser view
The system SHALL render a browsable list of all labs across every stage,
filterable by stage, difficulty, status, and topic tag.

#### Scenario: Filtering challenges
- **WHEN** the user filters the challenge browser by stage and difficulty
- **THEN** only matching labs are shown
- **AND** clearing the filters restores the full list

#### Scenario: Filtering by topic
- **WHEN** the user filters the challenge browser by a topic tag
- **THEN** only labs whose lab.yaml declares that tag are shown
- **AND** the topic filter's options list every tag used by any lab

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

### Requirement: Topology view
The system SHALL render a Topology page at `/topology` showing the active
lab's system as a diagram of nodes — the browser/host, the lab's
container, and its exposed ports — each node clickable through to more
detail about that part of the system, built only from the session file,
the lab's catalog metadata, and a read-only Docker status lookup.

#### Scenario: Topology with an active lab
- **WHEN** a lab is active and the user opens the Topology page
- **THEN** a diagram shows the lab's container as a node, connected to a
  node for each of its declared ports
- **AND** the container node's running/stopped status is shown, read live
  from Docker

#### Scenario: Clicking the container node
- **WHEN** the user clicks the container node
- **THEN** a detail panel shows the container's name, image, running
  status, and the exact `docker exec` command to get a shell in it
- **AND** no variant id, unused hint text, or any variants/ or solutions/
  content appears, the same rule as every other view

#### Scenario: Clicking a port node
- **WHEN** the user clicks a node for one of the container's exposed ports
- **THEN** its host:container port mapping is shown
- **AND** a link to open `http://localhost:<host-port>` is offered

#### Scenario: Topology with no active lab
- **WHEN** no lab is active and the user opens the Topology page
- **THEN** no diagram is shown
- **AND** the page prompts the user to start a lab from the roadmap or
  challenge browser, matching the dashboard's no-active-lab empty state

#### Scenario: Docker unreachable
- **WHEN** a lab is active but the Docker daemon is unreachable when the
  Topology page loads
- **THEN** the container node still renders from the session file
- **AND** its status is shown as unknown rather than the page failing

### Requirement: Web UI attack surface covers Docker-backed routes
Any local web UI route that reads live status from Docker, including the
Topology view, SHALL look up only the container name already recorded in
the session file (never a container name taken from the request), and
SHALL use a read-only Docker operation, consistent with the existing rule
that the web UI exposes only read routes and never executes a shell
command or writes to the filesystem.

#### Scenario: Reviewing the Topology route
- **WHEN** the security audit checks the Topology route's handler
- **THEN** the container name it passes to Docker comes only from the
  session file loaded server-side, never from a query parameter, form
  value, or other request input

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

### Requirement: Postmortems index view
The system SHALL render a Postmortems index page at `/postmortems`
listing every postmortem the user has written, draft and published
alike, filterable by stage, difficulty, and status (draft/published),
and SHALL link to it from the shared sidebar alongside Dashboard,
Roadmap, Challenges, and Topology.

#### Scenario: Viewing the postmortems index
- **WHEN** the user opens the Postmortems index page
- **THEN** every postmortem the user has written is listed, regardless
  of its publish flag
- **AND** each row links to that postmortem's existing detail page
  (`/postmortems/{slug}`)

#### Scenario: Filtering the postmortems index
- **WHEN** the user filters the postmortems index by stage, difficulty,
  or status
- **THEN** only matching postmortems are shown
- **AND** clearing the filters restores the full list

#### Scenario: No postmortems yet
- **WHEN** the user has not completed any lab's postmortem
- **THEN** the page shows an empty state prompting them to start a lab
  from the roadmap or challenge browser, matching the dashboard's
  no-active-lab empty state

#### Scenario: No variant or hint leakage
- **WHEN** the postmortems index renders
- **THEN** no variant id, unused hint text, or any variants/ or
  solutions/ content appears, the same rule as every other local UI view
