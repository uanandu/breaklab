## ADDED Requirements

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
