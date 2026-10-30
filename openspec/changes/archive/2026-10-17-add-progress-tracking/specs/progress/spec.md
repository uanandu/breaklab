# Spec Delta

## Purpose

Lets BreakLab remember what a learner has actually attempted and solved
across separate CLI invocations, so `breaklab status` and, later, the
public progress site and postmortems have a durable, committed record to
read from instead of only the transient per-attempt session state.

## ADDED Requirements

### Requirement: Attempt log
The system SHALL append every attempt to progress/progress.json with:
lab id, variant id, started_at, solved_at, failed_checks, hints_used,
postmortem path, and status (active, solved, complete, abandoned).

#### Scenario: Abandoned attempt
- **WHEN** the user stops or switches labs before solving
- **THEN** the attempt is saved with status abandoned

### Requirement: Status summary
The system SHALL show progress with `breaklab status`.

#### Scenario: Viewing status
- **WHEN** the user runs `breaklab status`
- **THEN** each roadmap stage shows completed labs out of total
- **AND** the active lab, if any, shows elapsed minutes and hints used
- **AND** the current daily streak is shown
