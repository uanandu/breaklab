# progress-site Specification

## Purpose
Turns the learner's own committed files — `progress.json`, lab
metadata, and published postmortems — into a static, shareable site,
without ever running anything the learner didn't explicitly ask for
(a push to `main`) or exposing anything they didn't explicitly mark
public (`publish: true`).

## Requirements

### Requirement: Static site build
The system SHALL generate a static site in site/ with `breaklab publish`,
using only progress.json, roadmap.yaml, lab metadata, and postmortems
marked publish: true. This flow is independent of hosted accounts: it
SHALL continue to work exactly as before for any user who never creates
an account, and remains available even for a user who does. The
`public-profile` capability's `/u/<name>` page is a separate, hosted
alternative — it does not replace or require `breaklab publish`.

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

#### Scenario: Works the same with or without an account
- **WHEN** a user who has created a hosted account and linked it via
  `breaklab login` runs `breaklab publish`
- **THEN** the static site build behaves identically to a user who has
  never created an account, reading only the local progress.json

### Requirement: Automated deploy
The repository SHALL include .github/workflows/publish.yml that rebuilds
the site and deploys it to GitHub Pages on every push to main.

#### Scenario: Push to main
- **WHEN** a commit with a new completed lab is pushed to main
- **THEN** the workflow runs `breaklab publish` and deploys site/
- **AND** the public page shows the new lab within one workflow run
