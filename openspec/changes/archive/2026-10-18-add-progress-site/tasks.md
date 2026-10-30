# Tasks

## 1. Reuse postmortem parsing for rendering

- [x] 1.1 In `internal/postmortem`, export the existing private section-splitting as `SplitIntoSections(body string) map[string]string` and the comment-stripping regex as `StripComments(text string) string` (behavior-preserving — `ValidateSections` keeps working by calling the newly-exported helpers instead of its private ones). Verify the existing `internal/postmortem` test suite still passes unchanged.

## 2. `internal/site`: data assembly

- [x] 2.1 Define `IndexView`, `StageView`, `CompletedLabView`, `PostmortemView`, and `SectionView` types, and implement `BuildIndex(cat *catalog.Catalog, pf *progress.ProgressFile, published map[string]bool) IndexView`: each stage gets its progress bar (`Completed`/`Total`/`Percent`, counting only `status == "complete"` per design.md Decision 2) and its completed labs' date/difficulty/minutes-to-fix (always present per Decision 1); a completed lab's `PostmortemHref` is set only when `published[attempt.Postmortem]` is true. Totals: `LabsCompleted` (complete-only), `VariantsSeen` (distinct lab+variant pairs across every attempt, any status — Decision 3), `Streak` (reusing `progress.Streak`). Verify with unit tests: a fixture with mixed `active`/`solved`/`abandoned`/`complete` attempts across two stages produces the right per-stage and total counts; a `complete` attempt whose postmortem isn't in `published` still shows its date/difficulty/minutes-to-fix but no href; `VariantsSeen` counts an abandoned attempt's variant too.
- [x] 2.2 Implement `CollectPostmortemPaths(pf *progress.ProgressFile) []string` (the `Postmortem` field of every `complete` attempt) and `LoadPublished(paths []string) (published map[string]bool, views map[string]PostmortemView, err error)`: for each path, read the file, split front matter, and if `publish: true`, parse it into a `PostmortemView` (lab/variant/date/sections, each section's paragraphs split on blank lines after stripping placeholder comments via the exported helpers from task 1.1) and record `published[path] = true`; a `publish: false` file is read but produces no view and no `published` entry; a missing or unreadable file is skipped rather than failing the whole load. Verify with unit tests: a `publish: true` fixture file appears in both maps with correctly split sections; a `publish: false` fixture file appears in neither; a missing path is skipped without error.

## 3. Build and templates

- [x] 3.1 Write `templates/site/style.css` (plain CSS, no external fonts/imports), `templates/site/index.html`, and `templates/site/postmortem.html` (both Go `html/template`, rendering `IndexView`/`PostmortemView`). Implement `site.Build(cfg Config) error` (labs dir, roadmap path, progress path, templates dir, output dir): wipe and recreate the output dir (design.md Decision 5), load the catalog and `progress.json`, collect and load postmortems (task 2.2), build the index view (task 2.1), render `index.html` and one `postmortems/<slug>.html` per published postmortem, and copy `style.css` into the output dir. Verify with an integration-style unit test using fixture `labs/`, `roadmap.yaml`, `progress.json`, and postmortem files in a temp dir: `site/index.html` and `site/postmortems/<slug>.html` exist with expected content for a published postmortem; an unpublished postmortem produces no page and no link; running `Build` again after flipping that postmortem's front matter to unpublished removes its now-stale page.
- [x] 3.2 Verify "nothing leaks" with a unit test: fixture labs include a hint string and variant/solution script content that must never appear anywhere under the built output directory, and no file path under `variants/` or `solutions/` is copied into it.
- [x] 3.3 Verify "works offline" with a unit test: the rendered `index.html` and any `postmortems/*.html` contain no `http://`/`https://` references and no `<script` tag, and `index.html`'s stylesheet reference is a relative path to the generated `style.css` (not an absolute URL).

## 4. CLI command

- [x] 4.1 Add `breaklab publish` to `cmd/breaklab/` (default paths: `labs`, `roadmap.yaml`, `progress/progress.json`, `templates/site`, `site`) and register it in `commands`/`commandOrder` in `cmd/breaklab/main.go`. Verify with a CLI-level test asserting `breaklab publish --help` exits 0 and is listed in `commandOrder`.
- [x] 4.2 Verify by hand against the real repo: solve `web-down`, create and validate its postmortem, hand-edit its front matter to `publish: true`, run `breaklab publish`, and confirm `site/index.html` shows the Linux fundamentals stage as complete with a working relative link to `site/postmortems/<slug>.html` whose five sections render; flip the front matter back to `publish: false`, run `breaklab publish` again, and confirm the page and link are both gone.

## 5. GitHub Actions deploy

- [x] 5.1 Write `.github/workflows/publish.yml`: on push to `main`, set up Go, `go build ./cmd/breaklab`, run `./breaklab publish`, then deploy `site/` to GitHub Pages via `actions/configure-pages` + `actions/upload-pages-artifact` + `actions/deploy-pages` (version tags, not pinned SHAs — design.md Decision 6), with the required `pages: write` / `id-token: write` permissions and a `github-pages` environment. Verify the YAML parses (`python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/publish.yml'))"` or equivalent) and that the required permissions, environment, and trigger are present.

## Workflow follow-up

- Run `gofmt -l` / `go vet` and the full `go test ./...` suite before archiving.
- Run `openspec validate add-progress-site --strict` and fix any findings.
- Confirm `site/` is still covered by the existing `.gitignore` entry and `templates/site/` / `.github/workflows/publish.yml` are trackable (not ignored).
- Archive the change with `/opsx:archive` once the above pass and the user has reviewed it.
