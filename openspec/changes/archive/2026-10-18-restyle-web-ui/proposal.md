## Why

The local web UI (`breaklab serve`) and the public progress site both
render with the browser's default look plus a handful of ad-hoc rules in
`templates/web/style.css` / `templates/site/style.css` — no consistent
color system, type scale, or component treatment. BreakLab is the tool a
learner opens every time they start or check a lab, and the thing a
public visitor sees on the progress page; a UI that reads as considered
rather than default-stylesheet reinforces the "this is a real incident
dashboard" premise the product is built around, the same way the
incident-brief framing and symptom-only tickets do. This change gives the
UI a deliberate visual design without touching any route, data flow, or
functional behavior.

## What Changes

- Introduce a small set of design tokens (CSS custom properties) for
  color, spacing, and type scale, shared by `templates/web/style.css` and
  `templates/site/style.css` so the local UI and the public site read as
  one product.
- Commit to a visual direction: a dark, terminal/incident-dashboard
  aesthetic (monospace accents for ids/commands, a muted dark background,
  a small accent palette for status/difficulty), with light-mode support
  via `prefers-color-scheme` rather than a toggle (no JS build step, no
  client-side state to persist).
- Restyle existing components with intentional, accessible treatment:
  status badges (available / in-progress / complete), difficulty
  indicators, the roadmap progress bars, tag chips (added in the topic-tag
  filtering change), the nav, and the dashboard's streak/totals summary.
  Status and difficulty get a text/shape distinction in addition to color
  so they don't rely on color alone.
- Restructure markup only where a component needs it to style correctly
  (e.g. wrapping a status string in a `<span class="badge badge-*">`) —
  no new routes, no new template data, no new HTMX endpoints.
- Keep the public site (`breaklab publish` output) fully self-contained:
  no external fonts, CSS, or scripts, so `site/index.html` still renders
  correctly opened from disk with no network.

## Capabilities

### New Capabilities
- `web-ui-design`: the design-system requirements for the local web UI
  and public site — shared tokens, theming (light/dark via
  `prefers-color-scheme`), and accessible (not color-only) status and
  difficulty treatment.

### Modified Capabilities
(none — `web-ui` and `progress-site`'s functional requirements, routes,
and data are unchanged; only presentation.)

## Impact

- `templates/web/style.css`, `templates/site/style.css` — rewritten
  around shared design tokens.
- `templates/web/*.html` (dashboard, roadmap, challenges, lab, postmortem)
  and `templates/site/*.html` — markup touched only where a component
  needs a wrapping element or class to style correctly.
- No changes to `internal/web/*.go`, `internal/site/*.go`, or any Go
  view-model struct — this is template/CSS only.
- No new dependencies: still Go `html/template` + HTMX + hand-written CSS,
  no JS build step, no SPA framework.
