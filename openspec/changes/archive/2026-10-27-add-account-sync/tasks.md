## 1. Data model: stable attempt identity

- [x] 1.1 Add `ID string` (`json:"id,omitempty"`) to `progress.Attempt`, generated via `crypto/rand` in `AppendActive`; verify `go test ./internal/progress/...` passes and a freshly appended attempt has a non-empty `ID` while old fixture files without the field still unmarshal cleanly
- [x] 1.2 Add a one-time local backfill (assign an `ID` to any attempt missing one) run at the start of `breaklab sync`; verify a `progress.json` fixture with ID-less entries gets IDs assigned exactly once and re-running is a no-op

## 2. Phase 1 — accounts exist, nothing reads them

- [x] 2.1 Add `internal/account` package with a `Store` interface (`CreateUser(email, username, password)`, `Authenticate`, `IssueToken`, `RevokeToken`, `DeleteUser`) and a SQLite-backed implementation using `modernc.org/sqlite`; verify unit tests cover signup (including the unique, `^[a-z0-9-]+$`-validated username), duplicate-email/duplicate-username/invalid-username rejection, and correct/incorrect login per the `account` spec scenarios
- [x] 2.2 Add `golang.org/x/crypto/bcrypt` for password hashing and `golang.org/x/crypto` + `modernc.org/sqlite` to `go.mod`; verify `go build ./...` succeeds with `CGO_ENABLED=0`
- [x] 2.3 Add session cookie issuance/validation (opaque `crypto/rand` token, `HttpOnly`/`Secure`/`SameSite=Lax`, DB-backed expiry) and API token issuance/revocation (stored as SHA-256 hash); verify unit tests cover session expiry and token revocation per the `account` spec
- [x] 2.4 Add `breaklab serve --db <path>` flag that constructs an `account.Store` and wires it into `web.Config.AccountStore` only when set; verify running `breaklab serve` with no `--db` leaves `AccountStore` nil and every existing web test still passes unmodified
- [x] 2.5 Add signup/login/logout/account-settings pages (`html/template` + HTMX, following the existing `templates/web` style) and their routes, mounted only when `AccountStore != nil`; verify an httptest-based test drives signup → login → generate token → revoke token → logout end to end
- [x] 2.6 Add account deletion (cascades sessions, tokens, synced attempts) wired to the settings page; verify a test confirms all three are gone after deletion and the account's `/u/<name>` (once built in phase 4) would 404

## 3. Phase 2 — sync exists, nothing else uses it

- [x] 3.1 Add `/sync/attempts` GET (returns terminal attempts for the authenticated account) and POST (`INSERT OR IGNORE` keyed on `(user_id, attempt_id)`), bearer-token authenticated; verify tests cover missing/invalid token (401), cross-account isolation, and idempotent re-POST per the `sync` spec
- [x] 3.2 Add `breaklab login --server <url> --token <token>` (both required) storing the server URL and token under `.breaklab/` (gitignored); verify a test confirms a missing flag exits non-zero with no I/O, and an invalid token stores nothing and leaves any existing stored server/token untouched
- [x] 3.3 Add `breaklab sync` (push terminal local attempts not yet pushed, pull terminal hosted attempts not yet local, merge by `ID`); verify an integration test round-trips attempts between two simulated local `progress.json` files through one hosted account and asserts no duplicates and no `active`-status attempt ever crosses the wire
- [x] 3.4 Verify `breaklab sync` never starts, stops, or queries Docker: add a test using a Docker fake that fails the test if any method on it is called during sync

## 4. Phase 3 — dashboard/roadmap become account-aware

- [x] 4.1 Add a merge helper that combines local `progress.json` attempts with an account's synced attempts by `ID` (local wins on conflict, since local is always at least as fresh as what it has already pushed); verify unit tests per the `web-ui` spec's new dashboard/roadmap scenarios
- [x] 4.2 Wire the dashboard handler to use the merge helper when the request carries a valid session and `AccountStore != nil`, else its existing local-only path unchanged; verify existing dashboard tests pass unmodified and a new test confirms a synced-only attempt (never run on this machine) affects totals/streak only when a session is present
- [x] 4.3 Wire the roadmap handler the same way; verify a new test confirms a lab's status reflects a synced attempt from another device, and that the exact same request with no session shows that lab as not-yet-attempted

## 5. Phase 4 — public profile

- [x] 5.1 Add an opt-in flag on the account settings page ("enable public profile") persisted in `account.Store`; verify a test confirms the flag defaults off
- [x] 5.2 Add `GET /u/{name}` rendering the same progress-summary shape as `breaklab publish`'s static site, built from the account's hosted attempts, returning 404 when the account hasn't opted in; verify tests cover opted-in, not-opted-in (404), and opt-out-then-404-immediately per the `public-profile` spec
- [x] 5.3 Verify no draft (`publish: false`) postmortem, variant script, solution script, or unused hint text ever appears on `/u/{name}`: add a security-style test mirroring the existing `internal/web/security_test.go` / `internal/site/site_security_test.go` pattern

## 6. Docs and regression

- [x] 6.1 Update `README.md`'s deployment section and add a note to `docs/deploy-raspberry-pi.md` distinguishing local/LAN `breaklab serve` (unauthenticated by design) from hosted mode (`--db`, requires real auth/TLS planning); verify the doc change is reviewed for accuracy against the `design.md` risk note
- [x] 6.2 Run the full suite (`go test ./...`) and confirm zero changes to any existing test's expected behavior outside the files touched by this change
