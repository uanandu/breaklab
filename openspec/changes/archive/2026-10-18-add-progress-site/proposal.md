# Proposal

## Why

Everything so far — `progress.json`, postmortems — lives only on the
learner's own machine. This is change 7 of 9 in the v1 build plan: it
adds `breaklab publish`, which turns those committed files into a
static, offline-capable `site/` a learner can look at or share, and a
GitHub Actions workflow that deploys it to GitHub Pages on every push to
`main`.

## What Changes

- New `internal/site` package (per `docs/breaklab-spec.md`'s own
  architecture: `site/site.go` is its own concern, not part of
  `internal/runner`): builds `site/` from `roadmap.yaml`, lab metadata
  (via the catalog), `progress/progress.json`, and
  `postmortems/*.md` whose front matter says `publish: true`.
- `breaklab publish`: wipes and rebuilds `site/` from scratch every run
  (so a postmortem that was published once and later flipped back to
  `publish: false` can't leave a stale page behind), writing
  `site/index.html`, one `site/postmortems/<slug>.html` per published
  postmortem, and a single `site/style.css`.
- `site/index.html` shows every roadmap stage with a progress bar, lists
  each stage's completed labs with date, difficulty, and minutes to fix,
  links each *published* postmortem to its own page, and shows totals:
  labs completed, variants seen, current streak.
- `templates/site/index.html`, `templates/site/postmortem.html` (Go
  `html/template`), and `templates/site/style.css` (plain CSS, copied
  verbatim — no external fonts, scripts, or stylesheets, so the site
  works when opened straight from disk).
- New `.github/workflows/publish.yml`: builds the `breaklab` binary, runs
  `breaklab publish`, and deploys `site/` to GitHub Pages on every push
  to `main`, using the standard `actions/configure-pages` +
  `actions/upload-pages-artifact` + `actions/deploy-pages` flow with
  version-tag references (commit-SHA pinning is change 9's job, per the
  build plan's own task list for that change).
- Out of scope: the local web UI (change 8) and the security audit's
  action-pinning pass (change 9).

## Capabilities

### New Capabilities

- `progress-site`: building the static `site/` from committed data only
  (`breaklab publish`) and deploying it to GitHub Pages on push to
  `main`.

### Modified Capabilities

_None._ `progress`'s existing requirements (attempt log, status summary)
and `postmortems`'s existing requirements (creation, required sections,
publishing control) keep their documented behavior — this change only
adds a new *reader* of `progress.json` and `postmortems/*.md`, so no
requirement text in either of their specs changes.

## Impact

- Affected code: new `internal/site/`; new `breaklab publish` subcommand
  in `cmd/breaklab/`; new `.github/workflows/publish.yml`.
- Affected data: new `templates/site/` (two `html/template` files, one
  CSS file); `site/` is generated output (already gitignored, per the
  existing `.gitignore`'s `site/` entry — not touched by this change).
- No new Go dependencies: postmortem bodies are turned into HTML by
  reusing `internal/postmortem`'s existing section-splitting (heading +
  paragraph text), not a markdown-parsing library — see design.md
  Decision 6.
- No impact on `labs/`, the CLI commands from changes 1–6, or the web
  UI — those are unaffected or come later.
