## ADDED Requirements

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
