## MODIFIED Requirements

### Requirement: Shared design tokens
The system SHALL define its visual design as two independent token
systems, one per surface, rather than one CSS-custom-property set
shared across both: the public progress site
(`templates/site/style.css`, `breaklab publish`, unaffected by the
`frontend` capability) keeps its existing shared-CSS-custom-property
model; the local app and hosted account pages (the `frontend`
capability's Astro/Astryx surfaces) use Astryx's own theme/token
system. Each surface SHALL still be internally consistent — color,
spacing, and type-scale values within a given surface come from that
surface's one token source, never a redeclared literal value.

#### Scenario: Token reuse across surfaces
- **WHEN** a color, spacing, or type-scale value is needed in
  `templates/site/style.css` (the public site) or on a
  `frontend`-capability page (the local app or a hosted account page)
- **THEN** it is read from that surface's own token source —
  `templates/site/style.css`'s shared custom properties for the
  public site, Astryx's theme system for `frontend` pages — never
  redeclared as a one-off literal value in either

#### Scenario: The two surfaces are no longer required to share one token set
- **WHEN** comparing a value (e.g. an accent color) between the public
  site and a `frontend`-capability page
- **THEN** they are not required to match — the two surfaces are
  deliberately independent design systems after this change, not one
  shared identity
