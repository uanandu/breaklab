## MODIFIED Requirements

### Requirement: Static site build
The system SHALL generate a static site in site/ with `breaklab publish`,
using only progress.json, roadmap.yaml, lab metadata, and postmortems
marked publish: true.

#### Scenario: Building the site
- **WHEN** the user runs `breaklab publish`
- **THEN** site/index.html shows every roadmap stage with a progress bar
- **AND** lists completed labs with date, difficulty, and minutes to fix
- **AND** links each published postmortem to its own page
- **AND** shows totals: labs completed, variants seen, current streak

#### Scenario: Nothing leaks
- **WHEN** a postmortem has publish: false
- **THEN** no page, link, or text from it appears anywhere in site/
- **AND** variant scripts, solutions, and hints are never copied into site/

#### Scenario: Works offline
- **WHEN** site/index.html is opened from disk
- **THEN** it renders fully with no external CSS, fonts, or scripts

#### Scenario: Building the postmortems index
- **WHEN** the user runs `breaklab publish`
- **THEN** a flat, sortable postmortems index page is generated listing
  every postmortem marked publish: true, across all stages
- **AND** each entry links to its existing per-postmortem page
- **AND** no postmortem with publish: false appears on it, the same
  rule as the rest of the site

#### Scenario: Postmortems index works offline
- **WHEN** the postmortems index page is opened from disk
- **THEN** it renders fully with no external CSS, fonts, or scripts
