# Design

## Context

This change audits changes 1–8 instead of adding a feature. Before
writing it, the code was surveyed directly:

- **Process execution:** 6 `exec.Command`/`CommandContext` call sites
  exist (`internal/docker`, `internal/runner` ×2, `internal/web/browser.go`
  ×3). There's no `sh -c` or `bash -c` anywhere, and all six use argument
  slices. The one shell-adjacent call is Windows' `cmd /c start` in the
  browser opener.
- **Ids:** the catalog already validates lab, variant, and stage ids
  against `^[a-z0-9-]+$` at load time, and the Docker wrapper re-checks
  every container/image name. Web routes look up ids and slugs in
  allowlists rather than building paths from them.
- **Gaps found:** ids and paths that come back from the two
  *hand-editable data files* skip validation. `.breaklab/session.json`'s
  `container`, and `progress.json`'s `lab`/`variant`/`postmortem`, are
  trusted as written (proposal.md lists the three concrete cases).
- **CI:** `.github/workflows/ci.yml` doesn't exist. `publish.yml` exists,
  with five tag-referenced `actions/*` steps.

See `proposal.md` for motivation and `specs/security/spec.md` for the
exact requirements.

## Goals / Non-Goals

**Goals:**
- Close the three data-file validation gaps.
- Remove the one shell-adjacent exec call.
- Create `ci.yml` with the checks `docs/breaklab-spec.md` describes plus
  `govulncheck`, and pin every action in both workflows to a real SHA.
- Turn each audit check into an automated test, so a later change can't
  silently undo it.

**Non-Goals:**
- Web UI authentication (an explicit v1 non-goal, even when LAN-bound).
- Anti-cheating. Variant details on disk stay on the honor system.
- Treating the learner as an attacker of their own CLI. An explicit path
  argument typed into `breaklab postmortem --validate <path>` has the
  same trust as `cat <path>` and is left as-is (Decision 4).

## Decisions

**1. The audit's checks become tests, not just a one-time read.**
An audit that's only a person reading code is true on the day it's done.
Each requirement gets a regression test:
- An AST scan of every non-test `.go` file under `internal/` and `cmd/`.
  It fails on any `exec.Command`/`CommandContext` call that passes a
  shell (`sh`, `bash`, `cmd`, `powershell`) with a `-c`/`/c` flag, or
  that builds an argument with `+` or `fmt.Sprintf`.
- A route-table test for the web UI.
- A default-bind test for `breaklab serve`.
- A `.gitignore` test.
- The existing "nothing leaks" and "works offline" site tests, extended
  to the new tampered-input cases.

The AST scan works on Go syntax, not text matching, so it doesn't
false-positive on comments or test fixtures.

**2. Windows' `cmd /c start <url>` is replaced with
`rundll32 url.dll,FileProtocolHandler <url>`.**
The current call isn't exploitable: it only runs when the host passes
`isLoopback`, so the URL can only ever contain a literal loopback IP or
`localhost`. But it is a shell invocation, so it either needs a written
exception in the audit or needs to go. `rundll32 url.dll,FileProtocolHandler`
is the standard shell-free way to open a URL on Windows, and the AST
check can then have zero exceptions. It's verified here by
cross-compiling only; there's no Windows host to run it on (see Risks).

**3. Session data is validated on load, rather than tightening the shared
id regex.**
The spec fixes the pattern at `^[a-z0-9-]+$`, which allows a leading
hyphen (`-f`, `--help`). Every container name the code *creates* is
`breaklab-<id>`, so it can never start with a hyphen. The only raw route
from disk to Docker is a hand-edited `session.json` `container`.
`LoadSession` will reject a session whose `lab` or `variant` fails the id
pattern, or whose `container` isn't exactly `breaklab-<lab>`, using the
same "corrupt — delete it to clear the session" error it already uses.
That closes the gap at its source and keeps the spec's regex word for
word. `catalog`'s private `validID` is exported as `catalog.ValidID` so
there's one definition of the pattern.

**4. Paths recorded in `progress.json` are confined to the postmortems
directory before they're read. An explicit CLI path argument is not.**
A shared helper, `postmortem.Confined(dir, recorded string) (string, error)`,
accepts a recorded path only if all of these hold:
- once cleaned and made absolute, it's a *direct* child of the
  postmortems directory;
- it ends in `.md`;
- it isn't a symlink (checked with `os.Lstat`).

It's used everywhere a path is read *because progress.json named it*:
`breaklab publish`, the no-path form of `breaklab postmortem --validate`,
and the web UI's postmortem viewer. Each caller handles a rejected path
the way it already handles a missing file: the site skips it, the web
UI returns 404, and validate reports an error. `site.Config` and
`web.Config` gain a `PostmortemsDir` field for this. `breaklab
postmortem` also validates an attempt's `lab`/`variant` with
`catalog.ValidID` before building a filename from them.

**5. `ci.yml` is created here, matching `docs/breaklab-spec.md`'s own
description, plus `govulncheck`.**
The build plan says to "add govulncheck to ci.yml", but no earlier change
created the file, so the Dependency audit requirement can't be met
without it. Its jobs follow the spec doc's "Testing and definition of
done" section:
- `gofmt -l` (fails if it prints anything), `go vet`, `go test ./...`
- the Docker lab integration test (`BREAKLAB_INTEGRATION=1`) on
  `ubuntu-latest`, which has Docker
- a `GOOS=linux GOARCH=arm64` build smoke check
- `go run golang.org/x/vuln/cmd/govulncheck@v1.9.0 ./...`

`govulncheck` is pinned to an exact version and isn't added to `go.mod`.
In its default mode it exits non-zero only on vulnerabilities whose
symbols are actually reachable from this module's code, which is exactly
the spec's "affects code reachable from main" bar. The workflow triggers
on `pull_request` and on push to `main`, uses `permissions: contents:
read`, and never uses `pull_request_target`.

**6. Every action is pinned to a full commit SHA, including GitHub-owned
`actions/*`, and SHAs are resolved from upstream, never written from
memory.**
Read literally, the spec only requires pinning *non-GitHub-owned*
actions. Both workflows use only `actions/*`, so a literal reading would
make the requirement vacuous. Tags on GitHub-owned actions are just as
mutable, and pinning costs nothing, so everything gets pinned, which
also satisfies the literal rule. Each SHA is resolved with `git
ls-remote https://github.com/actions/<name> "refs/tags/<tag>^{}"` (the
`^{}` peels an annotated tag to its commit), checked during the survey
to work from this sandbox. The tag is kept as a trailing comment, e.g.
`actions/checkout@<sha> # v4`, so it stays readable.

## Risks / Trade-offs

- **[Risk]** The Windows `rundll32` opener (Decision 2) can't be run
  here, only cross-compiled. → **Mitigation**: it's the same mechanism
  widely used Go browser-opening code relies on, and a failure is
  harmless: `breaklab serve` ignores opener errors, and the URL is
  always printed.
- **[Risk]** "CI's dependency audit step is green" can only be confirmed
  once the workflow runs on GitHub; this repo has no commits or remote
  yet. → **Mitigation**: the exact same `govulncheck` command runs
  locally during apply. That needs `vuln.go.dev` approved in the sandbox
  network policy (pending now). Until then that one check stays
  unverified locally and is called out.
- **[Trade-off]** Decision 3 rejects a hand-edited but otherwise working
  `session.json`. → **Mitigation**: it's gitignored, machine-local state,
  and the error says exactly how to recover (delete it).

## Spec clarifications worth flagging

1. **"CI SHALL run govulncheck" presumes a `ci.yml` no earlier change
   created.** Decision 5 creates it, using the CI the spec doc already
   describes. Flagging so it's clear this change adds the whole CI
   workflow, not one step.
2. **The pinning requirement only covers non-GitHub-owned actions, and
   none are used.** Decision 6 pins everything. A literal reading would
   leave both workflows on mutable tags while technically passing.
3. **The id regex allows a leading hyphen**, which matters for anything
   that becomes a CLI argument. Decision 3 closes the one path where
   that's reachable and keeps the regex word for word. Tightening it to
   `^[a-z0-9][a-z0-9-]*$` would be a reasonable follow-up if the spec
   allows changing it.
4. **"Every route in `internal/web/web.go`"**: the web package has no
   `web.go`; its route table is `routes()` in `internal/web/handlers.go`.
   The audit covers that function. Renaming the file to match is
   cosmetic and left alone.
5. **The spec scopes validation to "lab ids, variant ids, and stage ids"**,
   but the most practical traversal vector found was a *path* field
   (`progress.json`'s `postmortem`). Decision 4 treats it in the spirit
   of the "Path traversal attempt" scenario, even though it isn't an id.

## Open Questions

None that change the plan. Whether to tighten the id regex (item 3)
can be decided separately later.
