## Why

A full QA pass (black-box CLI/web testing plus a targeted code review) just
ran against the completed v1 build — all 9 prior changes, including
`add-security-audit`, are archived. Every feature and every security
guarantee held under test: the full lab lifecycle, random variant selection,
tiered hints, postmortem creation and validation, publish/site build, all
six web UI routes under path-traversal and injection fuzzing, session and
postmortem path tampering, and CI's action-pinning and `pull_request_target`
checks. No functional or security regression was found.

Two real gaps surfaced in how the existing guarantees stay true over time,
both proven during this session rather than hypothesized:

1. `internal/audit`'s workflow-pinning test (added by `add-security-audit`)
   only checks that each `uses:` line's SHA is 40 hex characters. It never
   checks that SHA still matches the tag named in its trailing comment, so a
   stale or mistyped SHA with a correct-looking comment passes silently.
2. `go.mod`'s `go` directive can drift behind the latest Go patch release
   with nothing flagging it — this happened for real just now. It was
   pinned to `go 1.26.1`; running `govulncheck` with that toolchain found
   25 reachable stdlib vulnerabilities (in `crypto/tls`, `crypto/x509`,
   `html/template`, `net`) already fixed in `go1.26.9`. Because CI's
   `govulncheck` job resolves its toolchain from `go.mod`
   (`go-version-file: go.mod`), this kind of drift would silently fail CI's
   security gate on PRs that never touched the vulnerable code.

## What Changes

- Add `.github/dependabot.yml` covering two ecosystems:
  - `github-actions`, which understands SHA-pinned `uses:` lines and
    proposes correctly-resolved re-pins against GitHub's own upstream data —
    closing gap 1 without hand-rolling a network-dependent `git ls-remote`
    check inside a unit test.
  - `gomod`, which proposes `go.mod` `go` directive bumps (including
    security-motivated ones) as new patches ship — closing gap 2.
- Add a regression test to `internal/audit` asserting `.github/dependabot.yml`
  exists and declares both ecosystems, so a later change can't silently
  remove this coverage (same pattern `add-security-audit` used for every
  other audit finding: turn it into a test, not just a one-time read).
- Already applied as part of this session's verification, carried forward
  by this change rather than left as a silent drive-by fix: bumped
  `go.mod`'s `go` directive from `1.26.1` to `1.26.9`, clearing the 25
  stdlib findings `govulncheck` reported.
- Small out-of-band cleanup, not a spec change: remove the Python-tooling
  entries (`.venv/`, `__pycache__/`, `*.pyc`, `.pytest_cache/`,
  `.ruff_cache/`) from `.gitignore` — leftover noise in a pure-Go project.
- Out of scope, confirmed not a bug: the web UI's lab detail page shows the
  hints-used count but never the revealed hint text itself. This matches
  `web-ui`'s current spec wording exactly ("hints used so far," not hint
  text) — not touched by this change. If the user wants revealed hint text
  surfaced in the web UI later, that's a separate, deliberate proposal
  against the `web-ui` capability, not a bug fix.

## Capabilities

### New Capabilities

_None._

### Modified Capabilities

- `security`: the "CI supply-chain hardening" requirement gains a freshness
  guarantee — pinned actions and the Go toolchain must have an automated
  path to staying current, not just a point-in-time audit.

## Impact

- Affected files: new `.github/dependabot.yml`; `go.mod` (`go` directive);
  `.gitignore`; a new test in `internal/audit`.
- No code behavior changes — this is CI/dependency-hygiene tooling plus the
  toolchain-version bump needed to make `govulncheck` pass today.
- No new Go dependencies.
