## Purpose

Surfaces which topic tags a user consistently needs more hints on, computed from their own attempt history, so a recurring blind spot becomes visible across runs instead of being lost once each individual lab is marked complete.

## ADDED Requirements

### Requirement: Per-tag hint average
The system SHALL compute, for each tag appearing on at least one lab with a complete attempt, the average hints_used across that tag's complete attempts, and the user's overall average hints_used across every complete attempt.

#### Scenario: Tag average computed from complete attempts only
- **WHEN** a tag's labs have both complete and non-complete attempts
- **THEN** only complete attempts count toward that tag's average

### Requirement: Pattern threshold
The system SHALL report a tag as a pattern only when it has at least 3 complete attempts AND its average hints_used is at least 1.5 times the user's overall average hints_used.

#### Scenario: Below sample-size floor
- **WHEN** a tag has fewer than 3 complete attempts
- **THEN** the tag is never reported as a pattern, regardless of its average

#### Scenario: Above threshold with enough samples
- **WHEN** a tag has at least 3 complete attempts and its average hints_used is at least 1.5x the overall average
- **THEN** the tag is reported as a pattern

### Requirement: Pattern visibility
The system SHALL show every current pattern tag in `breaklab status` and in the web dashboard, and show nothing in either place when there are no patterns.

#### Scenario: Status command shows patterns
- **WHEN** at least one tag qualifies as a pattern
- **THEN** `breaklab status` prints a patterns section listing each one

#### Scenario: No patterns yet
- **WHEN** no tag meets the pattern threshold
- **THEN** `breaklab status` and the web dashboard show no patterns section at all
