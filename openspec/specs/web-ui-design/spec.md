# web-ui-design Specification

## Purpose

Defines the shared visual design system for the local web UI and the
public progress site — color, theming, and accessible status/difficulty
treatment — so both surfaces read as one considered product rather than
default browser styling.

## Requirements

### Requirement: Shared design tokens
The system SHALL define its color, spacing, and type-scale values as a
shared set of CSS custom properties, used by both `templates/web/style.css`
and `templates/site/style.css`, so the local web UI and the public site
share one visual identity.

#### Scenario: Token reuse across surfaces
- **WHEN** a color, spacing, or type-scale value is needed in either
  `templates/web/style.css` or `templates/site/style.css`
- **THEN** it is read from a shared custom property rather than
  redeclared as a separate literal value in each file

### Requirement: Light and dark theming
The system SHALL render correctly in both light and dark color schemes,
following the visitor's OS-level preference (`prefers-color-scheme`),
with no theme toggle or client-side state required.

#### Scenario: Dark-preference visitor
- **WHEN** a visitor whose OS is set to dark mode opens the local web UI
  or the public site
- **THEN** the page renders with the dark color scheme on first paint,
  with no flash of unstyled or wrongly-themed content

#### Scenario: Light-preference visitor
- **WHEN** a visitor whose OS is set to light mode opens the local web UI
  or the public site
- **THEN** the page renders with the light color scheme on first paint

#### Scenario: Offline rendering unaffected
- **WHEN** `site/index.html` is opened from disk with no network
  connection, as required by the progress-site capability
- **THEN** theming still renders correctly, since it uses only inline or
  locally-served CSS and no external stylesheet or font

### Requirement: Status and difficulty are never color-only
Every place the UI signals lab status (available, in-progress, complete)
or difficulty, the system SHALL pair color with a text label or distinct
shape, so the signal does not depend on color perception alone.

#### Scenario: Status badge
- **WHEN** a lab's status is rendered anywhere in the local web UI or the
  public site (dashboard, roadmap, challenge browser, lab detail)
- **THEN** the status is shown as both a color and its text name (e.g.
  "complete"), never a color swatch alone

#### Scenario: Difficulty indicator
- **WHEN** a lab's difficulty is rendered anywhere in the local web UI or
  the public site
- **THEN** the difficulty is shown as both a color and a numeric or
  textual label, never a color swatch alone
