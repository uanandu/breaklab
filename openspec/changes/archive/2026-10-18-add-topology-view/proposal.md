## Why

Today the only way to see what's actually running for the active lab is
the one-line hint the CLI printed at `breaklab start` time (the
`docker exec -it breaklab-<lab-id> bash` command) and the dashboard's
plain-text "active lab" card. There's no visual sense of what you're
actually looking at — your machine, the lab container, which ports are
exposed and where they land. A small, honest system map turns "a
container is running somewhere" into something you can look at and click
through, the same way the roadmap and challenge browser turned "a list of
labs" into something browsable. This is scoped to what the architecture
actually has today (one container per active lab); it deliberately lays
out its data model so it keeps working, not redesigns itself, when
multi-container labs (`add-compose-labs`, already on the post-v1 roadmap)
land later.

## What Changes

- Add a new local web UI page, **Topology** (`/topology`), reachable from
  the nav, that renders the active lab's system as a small server-rendered
  diagram: you (the browser) → the host running `breaklab serve` → the
  lab's container → its exposed ports.
- Each node is clickable: the container node opens a detail panel (name,
  running/stopped status, image, and the lab's `docker exec` command —
  the same information already shown at `start` time, nothing new
  exposed); each port node shows its host:container mapping and a
  best-effort link to open it (`http://localhost:<port>`) — offered for
  every declared port, not just ones guessed to be HTTP, since lab.yaml
  doesn't declare a port's protocol.
- With no active lab, the page shows the same empty-state pattern the
  dashboard already uses: a prompt to start a lab from the roadmap or
  challenge browser, no diagram.
- Container status is read live via a new read-only `docker container
  inspect` call (container name comes only from the validated session
  file, never from the request) — the first place the local web UI reads
  from Docker rather than only from progress/catalog files. If Docker is
  unreachable, the node still renders from session data with its status
  shown as "unknown" rather than failing the whole page.
- No new write routes, no shell execution from a request parameter, no
  change to how labs are started/stopped/checked — this is a new,
  read-only view.

## Capabilities

### New Capabilities
(none)

### Modified Capabilities
- `web-ui`: adds a Topology view requirement (a new route, a new kind of
  live data — container status via Docker — and the empty-state rule for
  when no lab is active), and extends the "Web UI attack surface"-adjacent
  read-only guarantee to cover the new Docker-backed route.

## Impact

- `internal/web/`: new `topology.go` (handler + view-model), new
  `templates/web/topology.html`, nav link added to every local UI
  template.
- `internal/docker/docker.go`: new read-only `Inspect`-style method
  (container status/image), built the same way as the existing `Exists`
  call — `os/exec` with an argument slice, name validated against
  `^[a-z0-9-]+$` before use.
- `internal/runner/session.go`: no change — already the source of the
  active lab/container name; the topology handler reads it the same way
  the dashboard handler does.
- No change to `cmd/breaklab/`, `internal/catalog/`, `internal/progress/`,
  or any write path.
