## Context

See `proposal.md` for the why. Change 1 gave BreakLab a catalog; this
change makes a lab actually run. Reference: `docs/breaklab-spec.md`'s
"Spec: lab runner and random variants", "Lab format" (script contracts),
and "Starter labs" sections.

**For the DevOps learner reading this**: the runner is a small state
machine around one Docker container. `start` = build image, run container,
break it. `check` = run a script that tests the outcome. `reset` = start
again with the same breakage. `stop` = throw it away. The only memory
between commands is a tiny JSON file, `.breaklab/session.json`, the same
idea as a lock/pid file for a daemon.

## Goals / Non-Goals

**Goals:**
- Break and fix `web-down` end to end with real Docker.
- Every docker or script call uses `os/exec` with an argument slice.
- A clear exit code 2 and message when Docker is missing or stopped.
- The variant id stays hidden until `check` passes.

**Non-Goals:**
- Random/weighted variant choice, `--seed`, `BREAKLAB_FORCE_VARIANT` — change 3.
- `breaklab hint` — change 4.
- Writing `progress/progress.json` or recording solves/failures — change 5.
- Postmortem reminder — change 6.

## Decisions

**Docker via the `docker` CLI, behind a small interface.**
`internal/docker` exposes `Build`, `Run`, `Exec`, `Remove`, `Exists`, and
`Ping` on a struct that holds a command-runner function. Production code
passes `exec.Command`; unit tests pass a fake that records arguments. That
lets us test "no shell, args are slices, ids validated" without Docker,
and keeps the Docker SDK (a large dependency, awkward on a Pi) out of the
binary.

**One typed error for "Docker unavailable".** `Ping` runs
`docker info`; `exec.ErrNotFound` or a non-zero exit becomes
`docker.ErrUnavailable`. `main` maps it to exit code 2 and prints how to
start Docker (Docker Desktop on macOS/Windows, `systemctl start docker`
on Linux). Every runner command pings first, so the failure is the same
wherever it happens.

**Naming: image `breaklab-<lab>`, container `breaklab-<lab>`.** Both
derive from the lab id, which the catalog has already validated against
`^[a-z0-9-]+$`; the docker wrapper re-validates before building arguments
(defense in depth, cheap).

**Session file: `.breaklab/session.json`.**
`{ "lab": "web-down", "container": "breaklab-web-down", "variant":
"config-typo", "started_at": "..." }`, written atomically (temp file +
rename) with mode 0600. It lives under the working directory as the
conventions state, and is already gitignored. A missing file means "no
active lab"; a corrupt file is an error that names the file rather than
silently resetting.

**Variant for now: the first entry in `lab.yaml`.** The runner calls a
`pickVariant(lab)` function that returns `lab.Variants[0]`. Change 3
replaces only that function. The variant id is stored in the session but
printed only after a passing check.

**Applying the variant.** The script is copied to the container's `/tmp`
with `docker cp` and run with `docker exec <container> bash /tmp/<id>.sh`
as root (the default user). `docker cp` avoids bind mounts, so
`variants/` and `solutions/` never appear inside the container the learner
inspects, and no shell string interpolation is needed.

**Ports and check env.** `environment.ports` entries (`host:container`)
become `-p` arguments. For each, the runner sets
`BREAKLAB_PORT_<container port>=<host port>` for the check script, plus
`BREAKLAB_CONTAINER`. A non-numeric port is rejected when the lab is
loaded into the runner, not passed to Docker.

**Check scripts run per `check.run_in`.** `host`: `bash <lab>/check.sh`
with the env above. `container`: copy then `docker exec`. The first line
of stdout is the message; exit 0 = pass. A script that is missing is an
error naming the path, which is where the change 1 design said existence
would be checked. A 30-second timeout guards a hung check.

**Switching labs asks first.** If the session names a different lab,
`start` prompts `[y/N]` on stdin and stops the old container only on yes.
A non-interactive stdin counts as no. A `--yes` flag skips the prompt for
scripts and tests.

**Reset rebuilds the image.** The spec says "fresh container"; rebuilding
(cache-assisted) also picks up Dockerfile edits while authoring labs and
costs little. The session's variant is reused untouched.

**CLI wiring stays on stdlib `flag` + the dispatch map**, adding four
entries. Each command gets its own `FlagSet` with `--help`.

## Risks / Trade-offs

- [Running variant scripts as root inside a container the learner can
  also enter] → Mitigation: honor-system by design (spec non-goal "no
  anti-cheating"); scripts are copied in, not mounted, and removed after
  running so the obvious `ls /tmp` does not spoil the lab.
- [Host port 8080 already in use makes `docker run` fail] → Mitigation:
  surface Docker's error text plus a hint to stop whatever owns the port;
  no automatic port rewriting in this change.
- [`check.run_in: host` executes lab-supplied Bash on the user's machine]
  → Mitigation: labs are the user's own files; recorded for the change 9
  audit.
- [Tests that need Docker are slow and unavailable in some CI] →
  Mitigation: unit tests use the fake runner; one integration test is
  skipped unless `BREAKLAB_INTEGRATION=1`.

## Open Questions

- Whether `reset` should also re-apply the variant when the container
  survived (cheaper) or always recreate. Chosen: always recreate, for
  simplicity and to match the spec text.

## Spec feedback

Things in `docs/breaklab-spec.md` that read as unclear for this change —

- **Checking a fix** says the CLI "records a solve" and "reminds the user
  to write the postmortem", which depend on changes 5 and 6. The delta here
  intentionally omits both lines; they should be added back as `MODIFIED`
  scenarios when those changes land.
- **Not fixed yet** says "records a failed check attempt", with the same
  dependency on change 5; omitted likewise.
- The change plan says this change "uses the first variant only", yet the
  web-down variant scripts are listed under change 3. We ship only
  `config-typo.sh` (and its solution) here so the loop is real; the other
  two stay in change 3.
- The Reset and stop requirement is one sentence with no scenarios; two
  were added (variant retained, clean stop). Worth folding back into the
  build spec.
