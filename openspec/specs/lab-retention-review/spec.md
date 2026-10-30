# lab-retention-review Specification

## Purpose
Resurfaces a completed lab for an unaided re-solve once enough time has passed, so BreakLab can tell the difference between "solved once" and "still able to solve," rather than treating completion as permanent.

## Requirements

### Requirement: Review-due computation
The system SHALL compute a lab as review-due when it has a complete attempt and the most recent complete attempt's `solved_at` is more than the configured review interval in the past, without persisting any new field to progress.json.

#### Scenario: Lab becomes due after the interval
- **WHEN** a lab's most recent complete attempt's `solved_at` is older than the review interval
- **THEN** the lab is reported as review-due

#### Scenario: Recently completed lab is not due
- **WHEN** a lab's most recent complete attempt's `solved_at` is within the review interval
- **THEN** the lab is not reported as review-due

#### Scenario: Never-completed lab is never due
- **WHEN** a lab has no complete attempt
- **THEN** the lab is never reported as review-due

### Requirement: Review-due visibility
The system SHALL show which completed labs are review-due in `breaklab status`, the roadmap view, and the challenge browser, visually distinct from the available/in-progress/complete states.

#### Scenario: Status command lists due labs
- **WHEN** the user runs `breaklab status` and at least one completed lab is review-due
- **THEN** the output lists each review-due lab by id

#### Scenario: Roadmap and challenge browser show a review-due badge
- **WHEN** the roadmap or challenge browser renders a review-due lab's row
- **THEN** the row shows a "Review due" badge in addition to its complete status

### Requirement: Hints withheld during a review attempt
The system SHALL refuse to reveal a hint while the active attempt is a review attempt of an already-complete lab.

#### Scenario: Hint request during a review attempt
- **WHEN** the user requests a hint while their active attempt is a review attempt
- **THEN** the system refuses the hint and explains that hints are unavailable during a review attempt

### Requirement: Review completion resets the due date
The system SHALL record a completed review attempt as a normal attempt, so its `solved_at` becomes the baseline the next review-due computation is measured from.

#### Scenario: Completing a review attempt pushes the due date forward
- **WHEN** the user solves a lab during a review attempt
- **THEN** the lab is no longer review-due until the review interval elapses again from this attempt's `solved_at`
