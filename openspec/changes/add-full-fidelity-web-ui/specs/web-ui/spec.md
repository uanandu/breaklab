## MODIFIED Requirements

### Requirement: Dashboard view
The system SHALL render a dashboard showing the current streak, totals
(labs completed, variants seen), the active lab with elapsed time and
hints used if one is running, and the most recent postmortems. When a
lab is active, the dashboard SHALL also show that attempt's ticket
identifier, its declared port mapping and target scope, how many of its
available hints have been unlocked, and a tail of the container's own
log output. The dashboard SHALL also show real Docker engine version,
server uptime, average resolution time across solved attempts, and
postmortem draft/published counts, each read from data the system
already has or can query — never a fabricated or placeholder value.
When the request carries a valid hosted-account session, the dashboard
SHALL read/merge attempts from that account's hosted store in addition
to the local progress.json; with no session, it SHALL behave exactly
as before, reading only the local file.

#### Scenario: Dashboard with an active lab
- **WHEN** a lab is active and the user opens the dashboard
- **THEN** it shows the lab's elapsed time, hints used, and a reminder of the
  command to check it
- **AND** it shows that attempt's ticket identifier, port mapping, target
  scope, and "N of M hints unlocked" count
- **AND** it shows a tail of the active container's own log output

#### Scenario: Dashboard with no active lab
- **WHEN** no lab is active
- **THEN** the dashboard shows totals and streak only, with a prompt to start
  a lab from the roadmap or challenge browser

#### Scenario: Docker telemetry unreachable
- **WHEN** the Docker daemon is slow or unreachable while the dashboard
  loads
- **THEN** the Docker engine version, container log tail, and any
  resource-stat tiles degrade to an "unknown"/"—" placeholder rather
  than failing the page
- **AND** every other dashboard section (streak, totals, recent
  postmortems) still renders normally

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
complete), difficulty, and estimated minutes. Each lab with at least one
recorded attempt SHALL also show that attempt's ticket identifier. When
the request carries a valid hosted-account session, lab status SHALL be
computed from that account's merged (local + synced) attempts; with no
session, it SHALL behave exactly as before, using only the local
progress.json.

#### Scenario: Viewing the roadmap
- **WHEN** the user opens the roadmap page
- **THEN** stages appear in roadmap.yaml order
- **AND** each lab under a stage shows status, difficulty, and estimated minutes
- **AND** a completed lab links to its postmortem if one exists
- **AND** a lab with a recorded attempt shows that attempt's ticket identifier

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
filterable by stage, difficulty, status, and topic tag, and searchable
by a free-text query matched against lab title and tags. Results SHALL
be orderable by recent activity, difficulty, or alphabetical title. The
system SHALL offer a random-launch action that opens the detail page of
a random lab from the currently filtered set. When a lab is active, the
browser SHALL show a banner naming that lab and its ticket identifier.
The browser SHALL also show aggregate training-velocity statistics
(overall percent solved, a difficulty-curve breakdown by solved-lab
count, and median solve time) computed from the user's own recorded
attempts.

#### Scenario: Filtering challenges
- **WHEN** the user filters the challenge browser by stage and difficulty
- **THEN** only matching labs are shown
- **AND** clearing the filters restores the full list

#### Scenario: Filtering by topic
- **WHEN** the user filters the challenge browser by a topic tag
- **THEN** only labs whose lab.yaml declares that tag are shown
- **AND** the topic filter's options list every tag used by any lab

#### Scenario: Searching challenges
- **WHEN** the user enters free text in the challenge browser's search
  box
- **THEN** only labs whose title or tags match the text are shown,
  combined with any active stage/difficulty/status/topic filters

#### Scenario: Sorting challenges
- **WHEN** the user chooses a sort order (recent activity, difficulty,
  or alphabetical)
- **THEN** the filtered list is reordered accordingly without changing
  which labs match

#### Scenario: Launching a random challenge
- **WHEN** the user triggers the random-launch action
- **THEN** the system picks uniformly at random among the labs matching
  the currently active filters and search text
- **AND** opens that lab's detail page

#### Scenario: Active incident banner
- **WHEN** a lab is active and the user opens the challenge browser
- **THEN** a banner names the active lab and its ticket identifier and
  links to its detail page

### Requirement: Lab detail view
The system SHALL render a lab's ticket text and identifier, status,
hints used so far, and the exact CLI command to start or resume it, for
any lab the user selects from the roadmap or challenge browser. When the
lab is active, the system SHALL also show a tail of the container's own
log output and SHALL offer a live-updating stream of that same log
output. Each not-yet-revealed hint SHALL show a cosmetic point-value
label alongside its locked state; this value is for display only and is
never deducted from or added to any stored total.

#### Scenario: Opening a lab's detail page
- **WHEN** the user opens a lab from the roadmap or challenge browser
- **THEN** the ticket text, ticket identifier, status, and hints-used
  count are shown
- **AND** the CLI command to start or resume it is shown
- **AND** no variant id, unused hint text, or any variants/ or solutions/
  content ever appears, whether or not the lab is solved

#### Scenario: Active lab's diagnostic output
- **WHEN** the lab is active and the user opens its detail page
- **THEN** a tail of the container's own log output is shown
- **AND** a live-updating view of that same log output is available on
  the same page

#### Scenario: Locked hint shows a cosmetic cost only
- **WHEN** a hint has not yet been revealed
- **THEN** its locked state shows a point-value label
- **AND** no hint text, and no running total that the value has actually
  been subtracted from, is shown anywhere

### Requirement: Postmortem viewer
The system SHALL render any of the user's own completed postmortems in
the local web UI regardless of their publish flag, since that flag only
controls what becomes public. A fenced code block in a postmortem's
body SHALL render as a monospace block; a fenced block tagged `diff`
SHALL additionally color its added (`+`) and removed (`-`) lines.

#### Scenario: Viewing a local postmortem
- **WHEN** the user opens a completed lab's postmortem from the local UI
- **THEN** its full content renders, even when publish: false

#### Scenario: Postmortem with a diff block
- **WHEN** a postmortem's Fix section contains a fenced code block
  tagged `diff`
- **THEN** it renders as a monospace block with added lines and removed
  lines visually distinguished beyond color alone (e.g. a leading `+`/`-`
  character, which the diff syntax already provides)

## ADDED Requirements

### Requirement: Per-attempt ticket identifiers
Every recorded attempt SHALL have a stable, human-readable ticket
identifier derived from its position in the attempt log, shown
consistently everywhere that attempt is referenced (dashboard, roadmap,
challenge browser, lab detail, postmortem header). The identifier is a
display-layer value only — it introduces no new stored field and is
never used in place of the lab id or variant id for any Docker,
filesystem, or routing operation.

#### Scenario: Same ticket everywhere
- **WHEN** the same attempt is shown on two different pages (e.g. the
  dashboard's active-lab card and that lab's detail page)
- **THEN** both show the identical ticket identifier

### Requirement: Diagnostic log stream
The system SHALL expose a live-updating stream of the active session's
own container log output, scoped to exactly one lab's detail page at a
time. This route SHALL follow the same Docker-backed-route rule as
every other live Docker lookup in the UI: the container name comes only
from the session file loaded server-side, the operation is read-only,
and the stream stops and releases its underlying process when the
client disconnects or navigates away.

#### Scenario: Streaming the active lab's logs
- **WHEN** a lab is active and the user opens its detail page
- **THEN** new log lines the container produces appear on the page
  without a manual refresh

#### Scenario: Stream scoped to the active lab only
- **WHEN** the user requests the log stream for a lab id that does not
  match the active session's lab
- **THEN** the system does not open a stream and returns the same
  not-found response the rest of the lab detail routes use for an
  unknown or inactive lab

#### Scenario: Client disconnects
- **WHEN** the user navigates away from or closes the lab detail page
  while its log stream is open
- **THEN** the server stops following the container's logs and releases
  any process or resource it was using to do so, rather than continuing
  to run in the background
