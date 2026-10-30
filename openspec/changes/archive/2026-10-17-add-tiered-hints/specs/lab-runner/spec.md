# Spec Delta

## ADDED Requirements

### Requirement: Tiered hints
The system SHALL reveal one hint per `breaklab hint` call: general hints
first, then the active variant's hints, and count how many were used.

#### Scenario: Out of hints
- **WHEN** every hint has been shown
- **THEN** the CLI says so and suggests `breaklab reset`
