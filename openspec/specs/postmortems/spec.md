# postmortems Specification

## Purpose
Makes a learner explain what broke, how they found it, and how they
fixed it, because writing that down is what makes a fix stick and what
they'll reuse later — in a retro, a runbook, or an interview. A solved
lab only becomes "complete" once its postmortem passes validation.

## Requirements

### Requirement: Postmortem creation
The system SHALL create a postmortem file from templates/postmortem.md
for the most recently solved attempt and open it in $EDITOR.

#### Scenario: After a solve
- **WHEN** the user runs `breaklab postmortem` after a passing check
- **THEN** postmortems/<YYYY-MM-DD>-<lab-id>-<variant-id>.md is created
- **AND** its front matter holds lab, variant, stage, date, minutes to
  fix, hints used, and publish: false
- **AND** the file opens in $EDITOR (falling back to nano)

#### Scenario: Nothing solved
- **WHEN** no attempt has been solved since the last postmortem
- **THEN** the CLI exits non-zero with "Solve a lab first"

### Requirement: Required sections
Each postmortem SHALL contain these sections, each with at least
20 words of the user's own text: Summary, Detection, Root cause,
Fix, Prevention.

#### Scenario: Incomplete postmortem
- **WHEN** the user runs `breaklab postmortem --validate` on a file
  with an empty or placeholder Root cause section
- **THEN** the CLI lists each failing section and exits non-zero

#### Scenario: Valid postmortem
- **WHEN** every required section passes
- **THEN** the attempt is marked complete in progress.json

### Requirement: Publishing control
The system SHALL only publish postmortems whose front matter says
publish: true.

#### Scenario: New postmortems default to unpublished
- **WHEN** `breaklab postmortem` creates a new postmortem
- **THEN** its front matter sets publish: false
- **AND** nothing in the system ever changes that flag to true itself —
  only the learner, by hand-editing the file, controls publishing

### Requirement: Root cause consistency check
The system SHALL check, as an advisory (non-blocking) step of `breaklab postmortem --validate`, whether the Root cause section shares at least one meaningful word (case-insensitive, excluding common stopwords and words shorter than 4 characters) with the solved attempt's actual variant's `hints` text. This check SHALL NOT affect whether the postmortem is marked complete.

#### Scenario: Root cause mentions what actually broke
- **WHEN** the Root cause section shares at least one meaningful word with the solved variant's hints
- **THEN** `breaklab postmortem --validate` reports no warning for this check

#### Scenario: Root cause shares no words with what actually broke
- **WHEN** the Root cause section shares no meaningful word with the solved variant's hints
- **THEN** `breaklab postmortem --validate` prints a warning naming the mismatch, listing the words it expected to see
- **AND** the postmortem can still be marked complete if every required section otherwise passes its word-count check

#### Scenario: Warning is advisory only
- **WHEN** the root cause consistency check flags a mismatch
- **THEN** the CLI's exit code is unaffected by this check alone — only the existing required-sections check can cause a non-zero exit
