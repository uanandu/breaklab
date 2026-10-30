## Context

See `proposal.md` - Why/What Changes for motivation and scope. Relevant
current state: `internal/web.Config` is a plain struct of paths/funcs
passed into stdlib `net/http` handlers (`internal/web/server.go`,
`handlers.go`); `internal/progress` owns `progress.json` as the single
source of truth for a learner's `Attempt` history, with no stable
per-attempt identifier today (only its array index); `breaklab serve`
has no database, no sessions, and is explicitly unauthenticated by
design (`web-ui` spec, "Configurable bind host, still unauthenticated").

## Goals / Non-Goals

**Goals:**
- Add hosted accounts and cross-device sync without changing a single
  byte of behavior for a user who never creates an account.
- Keep the whole system buildable as one static Go binary (no cgo, no
  second runtime).
- Make the local/hosted boundary a single nil-check, not a fork of the
  codebase.

**Non-Goals:**
- Running labs anywhere but the learner's own machine (unchanged from
  today).
- Horizontal scaling or multi-region hosting — this is one instance,
  one SQLite file, sized for the same audience as this project already
  has (home-lab / Raspberry Pi scale).
- Social login, SSO, or any third-party auth provider.
- Resolving conflicting edits to the *same* attempt from two machines
  at once (see the "sync never moves active attempts" decision below,
  which removes the case where this could happen).

## Decisions

### One binary, one new flag, not a second binary
`breaklab serve` gains an optional `--db <path>` flag. Without it,
every handler behaves exactly as today (no import of `internal/account`
executes any code — `web.Config.AccountStore` is `nil` and every
account-aware branch short-circuits before touching SQLite). With it,
the server also mounts the account/sync/public-profile routes.
**Alternative considered:** a separate `breaklab-hosted` binary —
rejected, it doubles the build/release matrix for what is otherwise an
additive set of routes and one new package.

### Storage: SQLite via a pure-Go driver, behind a small interface
`modernc.org/sqlite` (no cgo, keeps `CGO_ENABLED=0` static builds).
Chosen over Postgres because this is a single hosted instance with
low write concurrency — sync is a periodic batch merge, not many
concurrent writers — and SQLite means zero additional infrastructure
to operate, consistent with this project's "lightweight, runs on a
Raspberry Pi" ethos. All access goes through a small `account.Store`
interface (`CreateUser`, `Authenticate`, `IssueToken`, `PushAttempts`,
`PullAttempts`, ...) so a future move to Postgres, if write volume ever
justifies it, replaces one implementation, not every call site.

### Attempts need a stable, portable identity to sync
`progress.Attempt` currently has no field that survives being copied
between machines — only its position in the local JSON array. Add
`ID string` (`json:"id,omitempty"`), a random 128-bit hex string
generated with `crypto/rand` (stdlib, no new dependency) when
`AppendActive` creates an attempt. Pre-existing attempts from before
this change have no `ID`; `breaklab sync` assigns one locally, once,
the first time it runs on a file with unidentified attempts, then
proceeds normally. The server's push endpoint does `INSERT OR IGNORE`
keyed on `(user_id, attempt_id)`, so re-pushing an already-synced
attempt is a no-op rather than a duplicate — sync can be safely re-run
or interrupted.

### Sync only ever moves terminal attempts
`breaklab sync` pushes and pulls attempts whose status is one of
`solved`, `complete`, `abandoned`, or `skipped` — never `active`. An
in-progress attempt only makes sense on the machine actually running
the lab (Docker state lives there too), so excluding `active` removes
the one case where two machines could disagree about the same
attempt's live state. This is simpler than any conflict-resolution
scheme and matches "labs always execute locally."

### Auth: hand-rolled sessions/tokens, bcrypt for passwords only
Cookie sessions (random `crypto/rand` token, `HttpOnly`, `Secure`,
`SameSite=Lax`, row in SQLite with an expiry) and API tokens (same
generation, stored as a SHA-256 hash, bearer-authenticated on
`/sync/*`) are simple enough to implement correctly with the stdlib.
Password hashing is the one place to not hand-roll: use
`golang.org/x/crypto/bcrypt` (a Go sub-repo, not a third-party
framework) rather than reimplementing a KDF.

### Dashboard/roadmap account-awareness is additive, not a rewrite
`internal/web.Config` gains an optional `AccountStore` field and
handlers gain one branch: "if the request has a valid session and
`AccountStore != nil`, merge in that account's synced attempts before
computing status; otherwise, do exactly what the function already
does." No existing local-only code path changes shape.

## Risks / Trade-offs

- **[Risk]** Hand-rolled session/token handling has a bug →
  **Mitigation**: short, fixed session TTL with sliding renewal;
  tokens and session ids are high-entropy random, never sequential;
  constant-time comparison for token lookup (hash in DB, compare
  hashes, not raw tokens).
- **[Risk]** A single SQLite file is a single point of failure for
  hosted data → **Mitigation**: out of scope to build backup tooling
  in this change; document that operators must back up the DB file
  (e.g. periodic copy, or `litestream` later) the same way they'd back
  up `progress.json` today.
- **[Risk]** Adding `ID` to `Attempt` touches a file some users may
  have external scripts around → **Mitigation**: the field is
  `omitempty` and purely additive; files without it still parse
  identically, and local-only behavior never reads or requires it.
- **[Risk]** Operators might point `--db` at a world-reachable deploy
  without realizing hosted mode is a different trust model than the
  existing "unauthenticated LAN" local mode → **Mitigation**: hosted
  mode's own docs section states plainly that it requires real
  authentication exposure planning (TLS, etc.), distinct from the
  existing local/LAN warning.

## Migration Plan

Four additive phases, each independently shippable and each leaving
every prior phase's behavior intact:

1. **Accounts exist, nothing reads them.** Ship `internal/account`,
   the SQLite schema, and the signup/login/logout/settings pages
   behind `breaklab serve --db <path>`. No CLI changes. Rollback:
   don't pass `--db`.
2. **Sync exists, nothing else uses it.** Ship `/sync/attempts`
   (GET/POST) and `breaklab login`/`breaklab sync`. Still fully
   additive — no existing route's behavior changes.
3. **Dashboard/roadmap become account-aware.** Add the merge branch
   described above, gated purely on session presence. Rollback: log
   out, or never create an account — behavior reverts exactly.
4. **Public profile.** Ship `/u/<name>`, 404 until an account opts in.

## Open Questions

- Exact session TTL and renewal policy (e.g. 30-day sliding window vs.
  fixed) — doesn't change the spec's observable contract ("a session
  is valid or it isn't"), safe to settle during implementation.
