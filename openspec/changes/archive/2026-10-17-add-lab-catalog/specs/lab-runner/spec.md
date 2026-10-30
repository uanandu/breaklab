## Purpose

Lets BreakLab discover, validate, and list the labs on disk so every
later command (start, check, hint, reset, stop) has a trustworthy
catalog of labs and roadmap stages to work from.

## ADDED Requirements

### Requirement: Lab catalog
The system SHALL load every `labs/<id>/lab.yaml` and `roadmap.yaml` at
startup and reject labs whose id, stage, or script paths are invalid.

#### Scenario: Listing labs
- **WHEN** the user runs `breaklab list`
- **THEN** labs are shown grouped by roadmap stage, in roadmap order
- **AND** each row shows id, title, difficulty, and status

#### Scenario: Invalid lab
- **WHEN** a lab.yaml references a stage missing from roadmap.yaml
- **THEN** `breaklab list` prints a warning naming the lab and the problem
- **AND** the other labs still load
