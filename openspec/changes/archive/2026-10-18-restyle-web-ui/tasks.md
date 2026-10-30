## 1. Shared design tokens

- [x] 1.1 Define the token set (`--bg`, `--bg-elevated`, `--fg`, `--fg-muted`, `--accent`, `--border`, spacing scale, font stack incl. a monospace accent face) as a `:root { ... }` block, and a `@media (prefers-color-scheme: light) { :root { ... } }` override, in `templates/web/style.css`
- [x] 1.2 Copy the identical token block into `templates/site/style.css` and verify the two blocks are byte-identical (`diff` the extracted `:root`/media blocks)
- [x] 1.3 Add a small Go test (e.g. `internal/web` or a new `internal/sitecheck`-style test) that reads both CSS files and fails if their token blocks diverge, so future edits can't drift silently — `internal/web/style_test.go`'s `TestSharedDesignTokensMatchAcrossStylesheets`
- [x] 1.4 Verify color contrast: text-on-background and each accent-on-background combination meets WCAG AA (4.5:1 text, 3:1 for the status/difficulty accents) in both color schemes — automated in `TestDesignTokensMeetWCAGAAContrast`, all pass (dark theme ratios ~6.6:1 to ~15.6:1)

## 2. Status and difficulty components

- [x] 2.1 Add `.badge` / `.badge-<status>` and `.difficulty` / `.difficulty-<n>` CSS rules using the shared tokens, each rendering a background/border color plus the existing text content (no color-only signal)
- [x] 2.2 Wrap `{{.Status}}` output in `templates/web/dashboard.html`, `roadmap.html`, `challenges.html`, and `lab.html` with the badge markup — dashboard.html's `DashboardView` has no `Status` field (only an active/inactive lab), so its active-lab card gets a `badge badge-in-progress` "active" indicator instead, matching the same badge component
- [x] 2.3 Wrap `{{.Difficulty}}` output in the same templates with the difficulty markup — `lab.html`'s `LabDetailView` has no `Difficulty` field, so only roadmap.html and challenges.html apply here
- [x] 2.4 Apply the same badge/difficulty markup to `templates/site/index.html` and `templates/site/postmortem.html` where status/difficulty are shown — `index.html`'s difficulty is wrapped; neither file has a status field (every listed lab is implicitly complete) and postmortem.html shows neither field, so nothing to wrap there
- [x] 2.5 Verify with `go test ./internal/web/... ./internal/site/...` that no view-model or route logic needed to change — only template markup — both pass, zero `.go` files touched for this change so far

## 3. Page-level restyle

- [x] 3.1 Restyle `templates/web/dashboard.html`'s totals/streak summary and active-lab card using the token set
- [x] 3.2 Restyle `templates/web/roadmap.html`'s stage sections and progress bars (`.bar` / `.bar-fill`) using the token set
- [x] 3.3 Restyle `templates/web/challenges.html`'s filter form, tag chips (`.tag`), and result rows using the token set, double-checking the topic-tag filter (stage/difficulty/status/tag) still renders and functions unchanged
- [x] 3.4 Restyle `templates/web/lab.html` and `templates/web/postmortem.html`, including the `<pre>` ticket/postmortem blocks, using the monospace accent face
- [x] 3.5 Restyle the shared `nav` element consistently across all local web UI pages
- [x] 3.6 Restyle `templates/site/index.html` and `templates/site/postmortem.html` to match, keeping all CSS inline or locally referenced (no external font/stylesheet/script)

## 4. Verification

- [x] 4.1 Run `breaklab serve` and manually check dashboard, roadmap, challenges (incl. filters), a lab detail page, and a postmortem page render correctly — verified visually in Chrome (dashboard, roadmap, challenges incl. `?tag=` filter, lab detail); no completed lab existed in the test fixture to exercise the postmortem page visually, but `go test ./internal/web/...` (`TestRealTemplatesRenderWithoutScriptTags`) exercises it against the real template
- [x] 4.2 Toggle the OS color scheme (or use browser devtools' `prefers-color-scheme` emulation) and confirm both local web UI and public site switch themes correctly with no unstyled flash — the browser automation tools available don't expose OS/devtools color-scheme emulation, so verified by overriding the token custom properties in-page to the light values on the real rendered markup (dashboard, lab detail, public site) and confirming legibility; the token values themselves and both themes' contrast are additionally checked by `TestDesignTokensMeetWCAGAAContrast`
- [x] 4.3 Run `breaklab publish`, then open the generated `site/index.html` directly from disk (`file://`) with network disabled, and confirm it still renders fully styled, per the progress-site "Works offline" scenario — the Chrome automation tool rejects `file://` navigation, so verified equivalently: grepped `site/index.html` and `site/style.css` for any `http(s)://` reference (none found — `href="style.css"` is the only stylesheet link, relative) and rendered the published output over a local-only static server to confirm it styles correctly with zero external requests
- [x] 4.4 Run `gofmt -l .`, `go vet ./...`, and `go test ./...` and confirm all pass with no non-template/CSS files changed outside what tasks 2–3 required — all green; the only non-template/CSS file touched is the new `internal/web/style_test.go` test file from task 1.3, no `.go` file was modified
