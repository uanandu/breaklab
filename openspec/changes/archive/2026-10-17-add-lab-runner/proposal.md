## Why

`breaklab list` can show labs, but nothing can run one yet. This is
change 2 of 9 in the v1 build plan: it adds the Docker wrapper and the
`start` / `check` / `reset` / `stop` commands so the core break-fix loop
works end to end on `web-down`, before variant randomization, hints,
progress, or any UI exist.

## What Changes

- `internal/docker` package: a thin wrapper that shells out to the
  `docker` CLI with `os/exec` argument slices (never a shell) for build,
  run, exec, remove, and inspect. A missing CLI or unreachable daemon
  surfaces as one typed error that the CLI maps to exit code 2 with a
  message explaining how to start Docker.
- `internal/runner` package: orchestrates start / check / reset / stop on
  top of the catalog and the Docker wrapper, and owns
  `.breaklab/session.json` (the active lab, its container name, and the
  chosen variant id).
- `breaklab start <lab>`: removes any previous container for that lab,
  builds the image, starts `breaklab-<lab>` with the lab's ports, runs the
  variant script inside it as root, and prints the ticket plus the
  `docker exec` command. The variant id is never printed. If a different
  lab is active, asks for confirmation before stopping it.
- `breaklab check`: runs the lab's check script (on the host or in the
  container per `check.run_in`) with `BREAKLAB_CONTAINER` and
  `BREAKLAB_PORT_<container port>` set, prints the script's one-line
  message, and on a pass reveals the variant id.
- `breaklab reset`: same variant, fresh container. `breaklab stop`:
  removes the container and clears the session.
- Lab data needed to make web-down runnable: `labs/web-down/check.sh`,
  and — because this change always uses the lab's **first** variant —
  `variants/config-typo.sh` with its `solutions/config-typo.sh`. The
  other two variants and the random selector are change 3.
- Missing-file errors for `check.script` and the variant script are raised
  here, at the point of use, as the change 1 design deferred them.
- Out of scope: random or weighted selection, `--seed`,
  `BREAKLAB_FORCE_VARIANT`, `breaklab hint`, `progress/progress.json`,
  recording solves or failed attempts, postmortem reminders, the web UI.

## Capabilities

### New Capabilities
_None._

### Modified Capabilities
- `lab-runner`: adds the **Starting a lab**, **Checking a fix**, and
  **Reset and stop** requirements. Because change 1 has not been archived
  yet, `openspec/specs/lab-runner/` may not exist when this is applied;
  the deltas are written as `ADDED` requirements so they merge cleanly
  either way (archive change 1 first to keep the order of the plan).
  Checking a fix is scoped down: it prints the message and reveals the
  variant, but "records a solve" and the postmortem reminder wait for
  changes 5 and 6.

## Impact

- Affected code: new `internal/docker/`, `internal/runner/`; new
  subcommands in `cmd/breaklab/` (`start`, `check`, `reset`, `stop`).
- Affected data: new `labs/web-down/check.sh`,
  `labs/web-down/variants/config-typo.sh`,
  `labs/web-down/solutions/config-typo.sh`; `.breaklab/session.json`
  created at runtime (already gitignored).
- New runtime requirement: the `docker` CLI and a running daemon for every
  command except `list`. No new Go dependencies.
- Security surface introduced here: process execution (docker, check
  scripts) and ids/paths flowing into container names and file paths —
  both governed by the existing `^[a-z0-9-]+$` and `os/exec` rules, and to
  be covered by the change 9 audit.
