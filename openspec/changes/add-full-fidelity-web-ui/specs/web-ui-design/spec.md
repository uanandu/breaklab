## ADDED Requirements

### Requirement: Locally-vendored typography
Any non-system font the design system uses (e.g. a monospace face for
code, tickets, and labels) SHALL be vendored as a local static asset
served by the application itself, never requested from a third-party
font host or CDN, so the local web UI and the public site keep the
offline/no-external-request guarantee the rest of this capability
already requires of its stylesheets.

#### Scenario: No network font request
- **WHEN** the local web UI or the public site renders any page that
  uses the vendored monospace font
- **THEN** the font is loaded from a path served by the application or
  present in the published static site, never from an external host
