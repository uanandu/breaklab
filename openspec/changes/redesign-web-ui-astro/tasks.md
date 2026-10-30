## 1. Scaffold the frontend toolchain

- [x] 1.1 Create `frontend/` with Astro initialized, pinned (no `^`/`~`) exact versions for `astro`, `react`, `react-dom`, `@astryxdesign/core`, `@astryxdesign/theme-neutral` (or chosen theme), `@astryxdesign/cli`, `@stylexjs/stylex`; commit `package-lock.json`; verify `npm ci && npm run build` succeeds and produces `frontend/dist/`
- [x] 1.2 Run `npx @astryxdesign/cli init` per Astryx's setup and confirm a minimal Astro page renders one Astryx component (e.g. a Button) as a hydrated island; verify by inspecting the built output for that island's hydration script and confirming every other page ships none
- [x] 1.3 Configure the Astro/Vite dev-server proxy so `/api/*`, `/sync/*`, `/signup`, `/login`, `/logout`, `/account/*`, `/u/*` forward to the Go backend; verify `npm run dev` + `breaklab serve` running side by side lets a browser hit only the Astro dev origin and still reach a Go-backed response
- [x] 1.4 Add `npm audit --omit=dev --audit-level=high` as a documented/scriptable check (e.g. an `npm run audit` script); verify it exits non-zero when run against a deliberately downgraded, known-vulnerable dependency and zero otherwise

## 2. Go-side plumbing shared by every migrated page

- [x] 2.1 Add `Config.FrontendDir` (path to `frontend/dist/`) alongside the existing `TemplatesDir`; verify a test confirms a Config with `FrontendDir` set serves a static asset from it and a Config without it leaves existing template-serving behavior untouched
- [x] 2.2 Add the "serve the same static shell file for any path-parameter value" routing helper (for `/labs/{id}`, `/postmortems/{slug}`, `/u/{name}`-shaped routes); verify a test requests two different ids against the same route and gets byte-identical HTML back, with the real id never required server-side for that response
- [x] 2.3 Establish the JSON API response convention (e.g. a small `writeJSON(w, v)` helper, consistent error-shape on failure) reused by every endpoint in sections 3-5; verify a unit test on the helper covers both success and error paths

## 3. Migrate the dashboard (proves the pattern end to end)

- [x] 3.1 Add `GET /api/dashboard` returning the same data `buildDashboard` already computes (streak, totals, active lab, ticket ID, Docker telemetry, recent postmortems), reusing `progressForRequest`/`currentUser` exactly as the current handler does; verify a test confirms identical underlying data to today's `dashboardHandler` for both a session and no-session request
- [x] 3.2 Build the Astro dashboard page consuming that endpoint, with no Astryx component yet required unless one fits naturally; verify `npm run build` produces the page and it renders real data when pointed at a running `breaklab serve`
- [x] 3.3 Wire `breaklab serve` to serve the new dashboard page at `/` instead of `dashboardHandler`'s template render; verify an end-to-end request to `/` returns the Astro build's HTML, not the old template output
- [x] 3.4 Delete `templates/web/dashboard.html` and `dashboardHandler`'s `html/template` rendering path (keep the new `GET /api/dashboard` handler); verify `go build ./...` and `go test ./...` still pass with the old template file gone

## 4. Migrate the remaining local-app pages

- [x] 4.1 Roadmap: add `GET /api/roadmap`, build the Astro page (including the per-stage skip action as a JSON `POST`), wire it in, delete `templates/web/roadmap.html` and the old handler's template path; verify an end-to-end test confirms stage skip still works and account-aware status still differs with/without a session, matching today's behavior
- [x] 4.2 Challenges + lab detail: add `GET /api/challenges` (filters/search/sort) and `GET /api/labs/{id}` plus `POST /api/labs/{id}/{start,check,hint,hypothesis,stop}`; build the Astro challenge-browser page and the lab-detail shell (path-parameter pattern from 2.2); delete `templates/web/challenges.html` and `templates/web/lab.html` plus their handlers; verify a full start→check→hint→stop cycle works end to end against the new API
- [x] 4.3 Topology: add `GET /api/topology`, build the Astro page (the graph rendering itself may still use the existing `cytoscape.min.js` vendored library, called from a small island or plain script); delete `templates/web/topology.html`; verify the graph renders real container/network data
- [x] 4.4 Postmortems: add `GET /api/postmortems` (index) and `GET /api/postmortems/{slug}`; build the Astro index page and detail shell; delete `templates/web/postmortem.html` and its index template; verify markdown/diff rendering still matches today's output and a nonexistent slug's API 404 surfaces a real "not found" UI state (design.md risk)
- [x] 4.5 Chaos + capstone: add their `GET /api/*` endpoints, build the two Astro pages, delete their templates/handlers; verify both render correctly including the capstone's locked/unlocked states

## 5. Migrate hosted account pages

- [x] 5.1 Signup/login: add `POST /api/signup` and `POST /api/login` (JSON-in, sets the same session cookie, JSON error on failure instead of a redirect+query-flash), build the two Astro pages; delete `templates/web/signup.html`/`login.html` and the old redirect-based handlers; verify signup→session-cookie-set→login-with-wrong-password-rejected still behave identically to the existing account tests
- [x] 5.2 Account settings: add `GET /api/account`, `POST /api/account/token`, `.../token/revoke`, `.../public-profile`, `.../delete` (JSON versions of the existing handlers); build the Astro page; delete `templates/web/account.html` and the old handlers; verify the existing delete-requires-username-confirmation behavior is preserved
- [x] 5.3 Public profile: add `GET /api/u/{name}` returning the `site.BuildIndex`-shaped JSON (still passing an empty "published" map per the existing no-postmortem-sync decision); build the Astro shell per the path-parameter pattern; delete `templates/web/public_profile.html` and the old handler; verify the existing not-opted-in/opt-out-immediately 404 tests still pass against the new endpoint

## 6. Cleanup and CI

- [x] 6.1 Remove now-unused shared `html/template` plumbing in `internal/web` (template cache, `_sidebar.html`, `renderPage`, `templates/web/style.css`) once no handler references it; verify `go vet ./...` reports no unused code and `grep -r renderPage internal/web` finds nothing
- [x] 6.2 Add a Node setup + `npm ci && npm run build` (including the audit gate from 1.4) step to `.github/workflows/ci.yml` before the Go build/test steps, and to any release workflow that builds the `breaklab` binary; verify CI passes on a clean checkout
- [x] 6.3 Update `README.md`'s dev-setup and deployment sections to describe the two-step build (`npm run build` in `frontend/`, then `go build`) and the dev-mode two-process workflow from task 1.3
- [x] 6.4 Run the full suite (`go test ./...`, `gofmt -l .`, `go vet ./...`) and confirm no regression outside files touched by this change
