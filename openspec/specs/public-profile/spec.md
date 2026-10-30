# public-profile Specification

## Purpose

Gives an account holder one dynamic, opt-in public page showing their
roadmap progress and published postmortems, as the hosted alternative
to building and deploying a static site themselves.

## Requirements

### Requirement: Opt-in public profile page
The system SHALL render a public page at `/u/<name>`, where `<name>` is
the unique username chosen at signup (`account` capability), for any
account that has opted in from account settings, showing the same
progress summary (streak, totals, roadmap stage progress) as `breaklab
publish`'s static site, and SHALL return 404 for any account that has
not opted in.

#### Scenario: Opted-in profile
- **WHEN** an account has enabled its public profile and a visitor
  requests `/u/<name>`
- **THEN** the page shows the account's roadmap progress and totals,
  built from its hosted (synced) attempts

#### Scenario: Not opted in
- **WHEN** an account has never enabled its public profile
- **THEN** `/u/<name>` returns 404

#### Scenario: Opting out removes the page immediately
- **WHEN** an account holder disables their public profile
- **THEN** `/u/<name>` returns 404 on the very next request

### Requirement: Only published postmortems ever appear
The system SHALL only show a postmortem on a public profile page when
it is marked `publish: true`, and SHALL never expose variant scripts,
solution scripts, or unused hint text through this page, the same rule
as the local web UI and the static site.

#### Scenario: Draft postmortem stays hidden
- **WHEN** an opted-in account has a postmortem marked `publish: false`
- **THEN** no title, link, or text from it appears on `/u/<name>`

#### Scenario: No variant or solution leakage
- **WHEN** any visitor views `/u/<name>`
- **THEN** no response on that page ever includes a variant id that
  hasn't been solved, a solution script, or hint text beyond what was
  already revealed
