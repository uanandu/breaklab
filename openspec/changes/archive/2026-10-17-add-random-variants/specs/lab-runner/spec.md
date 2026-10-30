# Spec Delta

## ADDED Requirements

### Requirement: Random variant selection
The system SHALL pick a variant at random, weighted by `weight`, and SHALL
not pick the variant used on that lab's previous attempt when the lab has
two or more variants.

#### Scenario: No immediate repeat
- **WHEN** web-down's last attempt used config-typo
- **AND** the user starts web-down again
- **THEN** the selected variant is not config-typo

#### Scenario: Reproducible selection
- **WHEN** the user runs `breaklab start web-down --seed 42` twice
- **THEN** the same variant is selected both times

#### Scenario: Forced variant for tests
- **WHEN** the BREAKLAB_FORCE_VARIANT environment variable is set
- **THEN** that variant is used and the selection rules are skipped
