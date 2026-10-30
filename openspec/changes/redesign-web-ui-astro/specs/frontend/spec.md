## Purpose

Lets the local app and hosted account pages be built with Astro and
Astryx's React components for a modern, product-grade look, without
turning BreakLab into something that needs a Node.js server process to
run — the Go binary stays the only thing that has to be running.

## ADDED Requirements

### Requirement: Static build, no Node runtime at request time
The system SHALL serve every in-scope page (dashboard, roadmap,
labs/challenges, topology, postmortems, chaos, capstone, signup, login,
account settings, public profile) from a static build produced ahead of
time by the Astro/Node toolchain, and `breaklab serve` SHALL respond to
every one of those routes without spawning or depending on a running
Node.js process.

#### Scenario: Serving an in-scope page needs no Node process
- **WHEN** `breaklab serve` is running and a client requests an
  in-scope route
- **THEN** the response is served from the pre-built static output
- **AND** no Node.js process is started or queried to produce it

#### Scenario: Build is a separate, prior step
- **WHEN** producing a release build or running locally for development
- **THEN** the frontend's static output is built once via the Node/npm
  toolchain (e.g. `npm run build`) before `breaklab serve` is started
- **AND** that build step is never invoked by the running Go binary
  itself

### Requirement: Islands-only hydration
The system SHALL ship zero client-side JavaScript by default on every
in-scope page, and SHALL hydrate a React island only for the specific
page regions that use an Astryx component; every other interactive
piece SHALL be plain Astro/vanilla markup and behavior.

#### Scenario: A page with no Astryx component ships no JS
- **WHEN** an in-scope page uses no Astryx component anywhere
- **THEN** its response includes no React runtime or island hydration
  script

#### Scenario: An Astryx component hydrates as an island
- **WHEN** an in-scope page includes an Astryx-powered component
- **THEN** only that component's region is hydrated as a React island
- **AND** the rest of the page remains static markup with no
  additional JS shipped on its account

### Requirement: JSON API backs every in-scope page
The system SHALL expose a JSON API under `internal/web` for every
in-scope page's dynamic data, authenticated the same way the page
itself is: a valid session cookie for account-aware data, same-origin
credentialed requests, no new auth mechanism introduced for this API.

#### Scenario: Fetching page data
- **WHEN** an in-scope Astro page or island loads
- **THEN** it fetches its data from the corresponding JSON API endpoint
  rather than receiving it embedded in server-rendered HTML

#### Scenario: Session-gated data parity
- **WHEN** a JSON API request carries a valid hosted-account session
  cookie, versus the same request with none
- **THEN** the response differs in exactly the way the equivalent
  `html/template` page used to (account-aware merge applied only with
  a session) — the move to JSON changes transport, not the
  session-gating behavior itself

### Requirement: Clean cutover, no dual rendering path
Once an in-scope page's Astro equivalent ships, the system SHALL remove
its old `html/template` template file and Go rendering code in the
same change — not keep both behind a flag or toggle.

#### Scenario: No leftover template for a migrated page
- **WHEN** an in-scope page has shipped as an Astro page
- **THEN** its old `templates/web/<page>.html` file and the Go handler
  code that rendered it no longer exist in the repository
