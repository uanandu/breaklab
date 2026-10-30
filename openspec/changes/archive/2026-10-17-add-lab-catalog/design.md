## Context

See `proposal.md` for the why. This change exists to give every later
change a trustworthy catalog of labs and roadmap stages — nothing here
touches Docker, checks, or the web UI. `openspec/specs/lab-runner/` does
not exist yet; this change creates it with only the Lab catalog
requirement. Reference: `docs/breaklab-spec.md`'s "Spec: lab runner and
random variants" and "Lab format" sections.

**For the DevOps learner reading this**: a "catalog" here just means "a
function that reads a folder of YAML files into Go structs and tells you
if any of them are broken." It's the same shape of problem as parsing
`docker-compose.yml` or a Kubernetes manifest — you're trusting a human
to write structured data by hand, so the loader's job is to catch
mistakes early and say exactly what's wrong, rather than let a typo
surface three commands later as a confusing crash.

## Goals / Non-Goals

**Goals:**
- Load and validate `roadmap.yaml` and every `labs/<id>/lab.yaml` at
  startup.
- Reject structurally invalid labs (bad id/stage format, unknown fields)
  without crashing the whole catalog load.
- `breaklab list` renders labs grouped by roadmap stage, in roadmap order.

**Non-Goals:**
- Running Docker, executing any script, or checking a fix — change 2.
- Picking a variant — change 3.
- Showing real progress/streak data — change 5. (See Decisions below for
  what `breaklab list` shows instead, in the absence of that data.)

## Decisions

**YAML parsing: `gopkg.in/yaml.v3` with strict decoding.**
The stdlib has no YAML support, and `yaml.v3` is the de facto standard
dependency for it in Go. By default it silently ignores unknown keys in
a YAML file, which would hide a typo like `stags:` instead of `stage:`
in someone's `lab.yaml`. Decoding through `yaml.Node` (or a decoder with
strict/`KnownFields` mode) makes an unknown key a load error instead of
a silent no-op — for a learner-facing tool, a loud error that names the
bad file beats a lab that mysteriously doesn't show up in `breaklab list`.

**Id/stage/script-path validation happens in the catalog loader, not
later.** The global convention (`^[a-z0-9-]+$` for ids) exists for
security reasons that matter starting in change 2 (these ids become
filesystem paths and Docker container names). Enforcing the same regex
here, in change 1, costs nothing and means a malformed id is caught at
`breaklab list` time with a clear message, not as a confusing failure
three changes later when it finally touches Docker.

**"Script paths are invalid" means shape, not existence — for now.**
The Lab catalog requirement says reject labs whose "script paths are
invalid." Taken literally as "file must exist on disk," this would break
`web-down` in this very change: its `lab.yaml` must still declare
`variants:` entries with `script: variants/config-typo.sh` (the schema
in "Lab format" requires it), but change 1 explicitly does not ship
those scripts — they arrive in change 3. The decision for this change is
that "invalid" means the path must be a relative path contained within
the lab's own directory (no `..`, no absolute paths, non-empty), i.e. a
shape check, not a filesystem existence check. Existence is checked
later, at the point a script is actually about to run (change 2 for
`check.script`, change 3 for variant scripts) — that's also the more
useful place for the error, since it's right before the thing that
needed the file. I flagged this ambiguity in the spec itself below.

**`breaklab list`'s "status" column shows `available` for every lab.**
The Listing labs scenario requires a status column, but progress data
(`progress/progress.json`) doesn't exist until change 5. Rather than
invent a fake status or block this change on progress tracking, catalog
and the list command treat "no progress data present" as "every lab is
available" — the same value `breaklab list` will show for any lab with
zero attempts once change 5 lands. No behavior here needs to change when
change 5 ships; it only adds more status values to choose from.

**CLI subcommand wiring: stdlib `flag` + a small manual dispatch table,
not Cobra, for now.** Change 1 only needs one subcommand (`list`).
Pulling in Cobra for that is premature; the stdlib's `flag` package plus
a `map[string]func([]string) int`-style dispatcher is enough and keeps
`go.mod` dependency-free apart from `yaml.v3`. This is pure
implementation detail — swapping to Cobra later, if `start`/`check`/
`serve`/etc. make manual dispatch awkward by change 8, changes no
externally visible behavior, so it doesn't need a spec change when it
happens (see Open Questions).

**Unknown-but-valid-shaped lab.yaml sections are parsed and stored, even
though change 1 doesn't use them.** `environment`, `check`, and the full
`variants` list (weights, hints) are part of the "Lab format" schema and
must round-trip through the loader even though nothing reads
`environment.dockerfile` or runs `check.script` until change 2. The
catalog's job is "parse the whole documented schema correctly," not
"parse only the fields this change happens to use."

## Risks / Trade-offs

- [Shape-only script-path validation could let a lab with a
  typo'd-but-shaped script path (e.g. `variants/cofnig-typo.sh`) pass
  `breaklab list` and only fail later, at `breaklab start` or
  `breaklab check` time in change 2/3] → Mitigation: acceptable for this
  change since the error still appears, just one command later than a
  full existence check would catch it; revisit if that proves confusing
  once change 2 lands.
- [Strict YAML decoding is stricter than most YAML tooling learners have
  seen, so an unrelated typo in a field this change doesn't even use
  could block `breaklab list`] → Mitigation: this is intentional — the
  error message names the file and the unknown key, turning a silent
  footgun into a one-line fix.

## Open Questions

- Whether to adopt Cobra (or stay on stdlib `flag` + manual dispatch) for
  CLI subcommands — can be decided whenever dispatch gets unwieldy,
  likely around change 8 (`serve` alongside `start`/`check`/`hint`/
  `reset`/`stop`/`status`/`postmortem`/`publish`). Purely an
  implementation choice; no spec or task depends on the answer.

## Spec feedback

Per the request for this change: things in `docs/breaklab-spec.md` that
read as wrong or unclear while implementing change 1 —

- The Lab catalog requirement's "reject labs whose ... script paths are
  invalid" doesn't say whether "invalid" means malformed or missing.
  Taken as "missing," it's inconsistent with the Change Plan's own
  sequencing (web-down's variant scripts don't exist until change 3).
  Resolved above as shape-only for now; worth stating explicitly in the
  spec text itself once this change is reviewed.
- The Listing labs scenario's "status" column implies progress state
  that doesn't exist until change 5. Not wrong, just worth a one-line
  note in the spec that status is `available`-only until progress
  tracking lands, so it's not re-litigated as a bug in change 5's review.
