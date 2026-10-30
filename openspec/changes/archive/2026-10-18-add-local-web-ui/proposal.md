# Proposal

## Why

`breaklab publish`'s public site is a snapshot, rebuilt by hand or by CI.
This is change 8 of 9 in the v1 build plan: it adds `breaklab serve`, the
view a learner opens every day — a live local dashboard, roadmap,
challenge browser, lab detail, and postmortem viewer, all read straight
from the files they already own, with no publish step and no gating on
`publish: true` (that flag only ever controlled what becomes public).

## What Changes

- New `internal/web` package (per `docs/breaklab-spec.md`'s architecture:
  `web/web.go` is its own concern, parallel to `site/site.go`): a
  `net/http` server rendering five views from the catalog, `progress.json`,
  and `postmortems/*.md`.
- `breaklab serve [--host] [--port]`: binds to `127.0.0.1:8787` by
  default, opens the user's default browser only when bound to a
  loopback address, exits with a clear port-in-use error instead of
  silently picking another port, and prints a one-time warning when
  bound to a non-loopback host (matching what `docs/deploy-raspberry-pi.md`
  already documents).
- Dashboard, roadmap, challenge browser (filterable by stage, difficulty,
  status — via plain HTML forms and full-page GET requests, not HTMX;
  see design.md), lab detail, and postmortem viewer, in
  `templates/web/*.html` (Go `html/template`, no JS build step).
- A real `available`/`in-progress`/`complete` status computation
  (`progress.ApplyStatus`, applied as a pass over an already-loaded
  catalog — see design.md for why it lives in `internal/progress`, not
  `internal/catalog`), replacing the change-1 stub that was explicitly
  left hardcoded to `"available"` pending progress tracking, which now
  exists. `cmd/breaklab/list.go` is updated to call it too, fixing
  `breaklab list`'s status column, which has shown `"available"` for
  every lab since change 1.
- `internal/site`'s private "latest complete attempt per lab" lookup is
  promoted to an exported `internal/progress` helper and reused by both
  `internal/site` and the new `internal/web`, instead of a third copy of
  the same logic.
- Out of scope: the security audit's formal hardening pass (change 9) —
  though every route here is built to its "no path built from unvalidated
  input, no arbitrary file reads" standard from the start, not as an
  afterthought change 9 has to fix.

## Capabilities

### New Capabilities

- `web-ui`: `breaklab serve` and its five views (dashboard, roadmap,
  challenge browser, lab detail, postmortem viewer), bound to loopback by
  default with no authentication at any bind address.

### Modified Capabilities

_None._ `lab-runner`'s **Lab catalog** requirement ("each row shows id,
title, difficulty, and status") never specified how `status` is computed,
so making it accurate instead of an always-`"available"` stub (see
design.md) is an implementation fix, not a requirement change — no text
in `openspec/specs/lab-runner/spec.md` changes.

## Impact

- Affected code: new `internal/web/`; `internal/progress/` (new
  `ApplyStatus` and exported "latest complete attempt" helpers);
  `internal/site/view.go` (refactored to use the latter);
  `cmd/breaklab/list.go` (calls `ApplyStatus` after loading progress);
  new `breaklab serve` subcommand in `cmd/breaklab/`. `internal/catalog`
  itself is unchanged.
- Affected data: new `templates/web/` (five `html/template` files plus a
  shared layout).
- No new Go dependencies: no HTMX, no markdown library — plain forms and
  the same section-splitting/front-matter helpers already built for
  postmortems and the public site.
- `docs/deploy-raspberry-pi.md` already exists and already documents
  this change's `--host`/`--port` behavior; no edits needed there unless
  implementation reveals a mismatch.
- No impact on the public site (change 7) or the CLI commands from
  changes 1–6 beyond the `computeStatus` fix.
