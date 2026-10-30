## Why

Today BreakLab is one user, one clone, one `progress.json`: there's no
way to see your streak/roadmap on a second machine, and the only
"publish" story is a static, read-only trophy case built per-fork via
`breaklab publish` → your own GitHub Pages. The user wants to move
toward a product-shaped app rather than a purely single-machine CLI:
hosted accounts so progress follows the learner across devices, and one
dynamic public profile instead of a GitHub Pages build step per fork.
This supersedes this project's prior "accounts, auth — out of scope for
v1" note; that was true before this pivot was decided and is
deliberately reversed here.

Labs themselves do not move: Docker execution stays entirely on the
learner's own machine through the existing `internal/runner`. Only
accounts and progress storage become a hosted, optional layer.

## What Changes

- Add a new `internal/account` package: users, password hashes
  (`bcrypt`), sessions, and API tokens, backed by SQLite (pure-Go
  driver, no cgo — consistent with the existing single static-binary
  build).
- Add hosted-only pages: landing/marketing, sign up, log in, log out,
  account settings (rotate/revoke API token, delete account). Built
  with the same `html/template` + HTMX stack as the existing local web
  UI — no SPA framework, no JS build step.
- Add a `/sync` API (`POST /sync/attempts`, `GET /sync/attempts`) that
  accepts/returns `progress.Attempt` records for the authenticated
  account, bearer-authenticated by the API token generated on the
  account settings page.
- Add `breaklab login --token <token>` (stores the token in
  `.breaklab/`, gitignored, alongside the existing session state) and
  `breaklab sync` (pushes local attempts not yet seen by the hosted
  account, then pulls any attempts recorded from another device into
  local `progress.json`).
- Dashboard and roadmap handlers gain an account-aware read path: when
  a request carries a valid session, they read/merge from the hosted
  store for that account; with no session, behavior is byte-for-byte
  what it is today, reading only the local file. **Local-only usage
  (clone, run, never sign up) keeps working unchanged, indefinitely —
  accounts are strictly opt-in.**
- Add one new public, opt-in page per account: `GET /u/<name>`,
  server-rendered from the hosted store. This is the dynamic
  replacement for `breaklab publish`'s static GitHub Pages output for
  any account that opts in; `breaklab publish` itself is untouched for
  users who never create an account.
- **BREAKING** (hosted deployment only, not the local CLI): running
  the web server in "hosted" mode now requires a SQLite database path
  and binds account-aware routes; this mode is new and opt-in at the
  deployment level, so existing `breaklab serve` (local, loopback,
  no accounts) is unaffected.
- New dependencies: a pure-Go SQLite driver (e.g.
  `modernc.org/sqlite` — no cgo, keeps the static-binary build) and
  `golang.org/x/crypto/bcrypt` (password hashing; this is a
  sub-repo of the Go project, not a third-party framework).

## Capabilities

### New Capabilities
- `account`: user signup/login/logout, sessions, API tokens, and the
  account settings page.
- `sync`: the `/sync` API and the `breaklab login`/`breaklab sync` CLI
  commands that move `progress.Attempt` records between a local
  `progress.json` and a hosted account.
- `public-profile`: the opt-in `/u/<name>` hosted page.

### Modified Capabilities
- `progress-site`: clarify that `breaklab publish`'s static-site flow
  is unaffected and remains available independent of accounts; add the
  cross-reference to `public-profile` as the hosted alternative.
- `web-ui`: dashboard/roadmap handlers gain the account-aware read path
  described above, while preserving today's no-session behavior
  exactly.

## Impact

- New package `internal/account` (users, sessions, tokens, SQLite
  schema/migrations).
- New routes on `internal/web`: `/signup`, `/login`, `/logout`,
  `/account`, `/sync/attempts` (GET/POST), `/u/{name}`.
- New CLI subcommands in `cmd/breaklab`: `login`, `sync`.
- New gitignored local state: an API token file under `.breaklab/`.
- New runtime dependency for hosted deployments: a SQLite database
  file/path (separate from `progress/progress.json`, which remains the
  local source of truth for local-only usage).
- `go.mod` gains two dependencies: a pure-Go SQLite driver and
  `golang.org/x/crypto`.
- Deployment docs (`docs/deploy-raspberry-pi.md`) gain a note that
  "hosted mode" is a separate, opt-in deployment shape from the
  existing local/LAN `breaklab serve` story.
