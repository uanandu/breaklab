## Why

The local web UI and the hosted account pages (from `add-account-sync`)
currently look like a utility tool — server-rendered Go `html/template`
pages with hand-rolled CSS. The user wants BreakLab's app surface to
feel like a real, modern SaaS product, using Astryx
(https://astryx.atmeta.com, a React 19 + StyleX component library) for
that visual language. Astryx is React-only, so getting its actual
components — not just copying its look by hand — means adopting Astro
as the page framework, with React islands wherever an Astryx component
is used.

## What Changes

- Add a new Astro project (new top-level `frontend/` directory) that
  renders every in-scope page: dashboard, roadmap, labs/challenges,
  topology, postmortems, chaos, capstone, and the hosted account pages
  (signup, login, account settings, public profile). Pages ship zero
  client JS by default; a React island is mounted only where an Astryx
  component is actually used.
- **BREAKING** (internal, not user-facing): `internal/web`'s page
  handlers stop rendering `html/template` output for every in-scope
  route. Each becomes a JSON API instead (e.g. `GET /api/dashboard`,
  `GET /api/roadmap`), consumed by the corresponding Astro page/island
  client-side. The existing cookie-session auth
  (`internal/account`/`sessionCookieName`) is reused as-is for these
  calls — no new auth mechanism.
- `breaklab serve` serves the Astro build's static output (HTML/JS/CSS)
  instead of executing Go templates for in-scope routes. No Node.js
  process runs at request time — Node/npm are build-time-only tools,
  the same role `templates/web/js/htmx.min.js` being vendored plays
  today, just produced by a real build instead of a hand-copied file.
  This preserves the project's existing "ships as one statically-linked
  binary, no other language runtime required" constraint.
- `templates/web/*.html` and the Go rendering code for every in-scope
  page are deleted outright once the Astro equivalent ships — no
  feature flag, no dual-maintained rendering path (per this project's
  existing convention of deleting dead code rather than leaving unused
  paths around).
- Out of scope, explicitly unchanged: `breaklab publish`'s static site
  (`internal/site`, `templates/site/*`) stays exactly as it is —
  server-less, offline-capable, zero JS runtime. It is not part of this
  redesign.
- New build-time dependencies: Astro, React 19+, `@astryxdesign/*`
  packages, `@stylexjs/stylex`. Dependency hygiene (pinned exact
  versions, committed lockfile, an audit step in CI) is a project
  discipline decision recorded in design.md — it applies regardless of
  framework choice and isn't a reason to avoid Node/npm, which no
  modern frontend tooling avoids at build time.

## Capabilities

### New Capabilities
- `frontend`: the Astro + React-islands build for the local app and
  hosted account pages — its build/serve model (static output, no
  Node at request time, islands-only hydration) and the JSON API
  surface `internal/web` exposes to it.

### Modified Capabilities
- `web-ui-design`: the "shared design tokens" requirement currently
  names `templates/web/style.css` and `templates/site/style.css`
  explicitly as the two surfaces sharing one token set. Once the local
  app moves to Astro/Astryx's own theme system, that's no longer true —
  there are now two independent design systems (Astryx's, for the
  redesigned surfaces; the existing CSS-token system, for the
  unaffected public site). The dark/light theming and
  never-color-only-status requirements continue to apply to both, by
  each system's own means.

No change to `web-ui`, `account`, or `public-profile`'s existing
behavioral requirements: every requirement in those specs (e.g. "the
dashboard shows the current streak", "the system SHALL let a visitor
create an account...") is written at the level of observable behavior,
not rendering technology, and remains true under this change. This
redesign changes *how* those pages are built and *what they look
like*, not what they guarantee.

## Impact

- New top-level `frontend/` directory: Astro project, React islands,
  Astryx components, `package.json`/lockfile, build config.
- `internal/web`: most handlers change from `html/template` rendering
  to JSON responses; `templates/web/*.html` deleted for every in-scope
  page once its Astro equivalent ships.
- `cmd/breaklab/serve.go`: serves the Astro build's static output
  (`frontend/dist/` or equivalent) for in-scope routes.
- CI (`.github/workflows/*`) and build docs: a new frontend build step
  (`npm ci && npm run build`) runs before `go build`, for both local
  dev and release builds.
- `internal/site`/`templates/site/*`/`breaklab publish`: untouched.
- `go.mod`: unaffected — no new Go dependency, since the Go side only
  needs to serve static files and JSON, both stdlib `net/http`.
