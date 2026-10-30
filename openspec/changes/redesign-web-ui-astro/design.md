## Context

See proposal.md for motivation/scope. Relevant current state: `internal/web` renders every in-scope page server-side with `html/template` against live data (progress.json, the lab catalog, Docker telemetry, account/session state) on every request — there is no build step in that loop today; a learner adds a lab or writes a postmortem and sees it immediately. `internal/account` already provides cookie-session auth (`sessionCookieName`) and bearer-token auth (`/sync/*`), both of which this change reuses as-is. CI (`.github/workflows/ci.yml`) currently only runs `gofmt`/`go vet`/`go test`/Docker integration tests — no Node/npm step exists anywhere in this repo yet.

## Goals / Non-Goals

**Goals:**
- Every in-scope page gets Astro-built static output + Astryx-powered React islands, with no Node.js process required to serve a single request in production.
- Adding a lab, writing a postmortem, or creating an account never requires rebuilding the frontend — exactly the same "no build step in the loop" property the app has today.
- The Go binary remains the only thing that has to be running.

**Non-Goals:**
- Server-side rendering (SSR) of React/Astro at request time — that would require a Node runtime alongside Go, which directly contradicts the single-binary constraint. Not pursued.
- Enumerating every lab/postmortem/username as a separate static page at build time (Astro's `getStaticPaths`) — rejected below; it would silently 404 anything created after the last frontend build.
- Matching the public site's (`templates/site`) visual identity. They are intentionally becoming two independent design systems (see the `web-ui-design` delta).

## Decisions

### Astro builds shells, not content — all record data stays client-fetched
For any route with a path parameter (`/labs/{id}`, `/postmortems/{slug}`, `/u/{name}`), Astro generates exactly **one** physical static HTML file (a placeholder build-time param value is fine — it's discarded). Go's existing path-parameterized routes (`GET /labs/{id}`, etc.) serve that same static file for *any* value of the parameter; the real id/slug/name is read from `window.location.pathname` client-side and used to call the JSON API. No page's catalog/progress/account content is ever baked in at `npm run build` time — it's all fetched at runtime, the same "always live" property the app has today. **Alternative considered:** `getStaticPaths` enumerating every known lab/postmortem at build time — rejected because a lab or postmortem added after the last frontend build would 404 until someone remembered to rebuild, a real regression from today's always-live behavior.

### Islands are per-component, not per-page
A page is plain Astro (zero JS) unless it contains an Astryx component, in which case only that component's DOM region hydrates as a React island (Astro's standard `client:load`/`client:visible` directive). A page needing live data but no Astryx component (e.g., a plain progress bar) can still fetch via a small vanilla `<script>` — React is reserved for Astryx's own components, not used as a general templating tool.

### JSON API: additive, same host, same auth
Every in-scope page gets a matching `GET /api/<page>`-shaped endpoint (exact paths decided per-page during implementation; e.g. `/api/dashboard`, `/api/roadmap`, `/api/labs/{id}`) returning the same data the `html/template` view used to receive, as JSON. These live alongside the existing `/sync/*` and account routes in `internal/web`, reuse `currentUser`/`requireSession`/`progressForRequest` exactly as today, and are served from the same origin as the static build — so there's no CORS surface in production.

### Dev workflow: Astro dev server proxies to Go
`npm run dev` (Astro's dev server, e.g. port 4321) proxies `/api/*`, `/sync/*`, `/signup`, `/login`, etc. to the Go backend (port 8787) via Astro/Vite's dev-server proxy config, so the browser only ever sees one origin even in development — avoiding any need for CORS headers in the API itself, in dev or prod.

### Dependency hygiene (the earlier Node/npm security concern)
`frontend/package.json` pins exact versions (no `^`/`~` ranges) for every direct dependency (Astro, `react`, `react-dom`, `@astryxdesign/*`, `@stylexjs/stylex`), commits `package-lock.json`, and CI runs `npm audit --omit=dev --audit-level=high` (failing the build on a high/critical advisory) alongside `npm ci` (which refuses to install if the lockfile and `package.json` disagree). This is a dependency-hygiene practice, not a framework-specific mitigation — it would apply identically if the framework were Svelte or anything else.

### Migration is page-by-page, each page's cutover is immediate
Pages migrate one at a time (see tasks.md for order). The moment a page's Astro version ships and is wired into `breaklab serve`, that page's `templates/web/<page>.html` and its Go `html/template`-rendering code are deleted in the same change — never kept behind a flag "for now." This matches the proposal's "clean cutover" decision and this project's existing convention of deleting dead code rather than accumulating unused paths.

## Risks / Trade-offs

- **[Risk]** This is the largest rewrite BreakLab's web layer has seen — nearly every `internal/web` handler changes shape → **Mitigation**: migrate and ship one page at a time (tasks.md), each with its own passing tests, rather than one big-bang PR; the app stays fully working (old pages) throughout the migration.
- **[Risk]** The "serve the same static file regardless of the path parameter" pattern means Go's HTML layer never 404s a genuinely nonexistent lab id/slug/username — only the JSON API does → **Mitigation**: every such page's client-side code must handle a 404 JSON response with a real "not found" UI state, not a silent blank page; call this out explicitly in that page's migration task.
- **[Risk]** npm supply-chain exposure (the concern raised earlier in conversation) → **Mitigation**: pinned versions, committed lockfile, `npm audit` gate in CI, minimal direct dependency count — see the Dependency hygiene decision above.
- **[Risk]** Astro/React/Astryx is an entirely new toolchain this repo has no prior experience operating → **Mitigation**: fully isolated under `frontend/`; the Go side only ever shells out to `npm ci && npm run build` as an opaque build step, so no Go code needs to understand the JS toolchain's internals.
- **[Risk]** Two concurrently-active OpenSpec changes now both touch `web-ui`-adjacent specs (`add-full-fidelity-web-ui`, which added ticket IDs/Docker telemetry to the `html/template` dashboard/roadmap, and this one, which replaces that rendering entirely) → **Mitigation**: not resolved here — flagged for the user to sequence (likely: finish/archive `add-full-fidelity-web-ui` first, or fold its still-relevant content requirements into this migration's per-page tasks so they aren't silently dropped).

## Migration Plan

1. **Scaffold + prove the pattern on one page.** Create `frontend/` (Astro + React + Astryx + StyleX, pinned deps, lockfile), the Go static-serving plumbing (`Config.FrontendDir`, catch-all-per-shell routing), and the dev-proxy config. Migrate the dashboard — the simplest page — end to end: `GET /api/dashboard` JSON endpoint, Astro dashboard page/shell, delete `templates/web/dashboard.html` and its Go rendering code. This page proves the whole pattern before scaling it.
2. **Migrate the remaining local-app pages**, one at a time: roadmap, challenges/labs (incl. the lab detail shell and its start/check/hint/stop actions — these become POST JSON calls too), topology, postmortems (index + detail shell), chaos, capstone.
3. **Migrate the hosted account pages**: signup, login, account settings, public profile shell.
4. **Remove leftover shared plumbing**: once no page references them, delete the shared `templates/web/style.css`/`_sidebar.html`/template-caching code in `internal/web` that only existed to serve `html/template` output.
5. **CI/release**: add a Node setup + `npm ci && npm run build` (with the `npm audit` gate) step before `go build` in `ci.yml` and any release workflow; update `README.md`'s dev-setup instructions.

Rollback at any point before step 4 is simply "don't flip `breaklab serve` to the new route for that page yet" — each page's old template stays intact until its own migration task explicitly deletes it.
