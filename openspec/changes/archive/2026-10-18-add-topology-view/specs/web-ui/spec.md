## ADDED Requirements

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
