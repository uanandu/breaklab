# Spec Delta

## Purpose

Turns the learner's own committed files — `progress.json`, lab
metadata, and published postmortems — into a static, shareable site,
without ever running anything the learner didn't explicitly ask for
(a push to `main`) or exposing anything they didn't explicitly mark
public (`publish: true`).

## ADDED Requirements

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

### Requirement: Automated deploy
The repository SHALL include .github/workflows/publish.yml that rebuilds
the site and deploys it to GitHub Pages on every push to main.

#### Scenario: Push to main
- **WHEN** a commit with a new completed lab is pushed to main
- **THEN** the workflow runs `breaklab publish` and deploys site/
- **AND** the public page shows the new lab within one workflow run
