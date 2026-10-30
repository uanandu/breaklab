## 1. internal/web: postmortems index view

- [x] 1.1 Add `PostmortemsIndexView` and a row type (slug, lab title,
      stage, difficulty, solved date, status draft/published, detail
      href) to `internal/web/postmortem.go`
- [x] 1.2 Add `buildPostmortemsIndex(cat *catalog.Catalog, pf
      *progress.ProgressFile, published map[string]bool, q url.Values)
      PostmortemsIndexView`, filtering by `stage`/`difficulty`/`status`
      query params (status: `draft` = complete but not published,
      `published` = publish: true) and sorting by solved date, most
      recent first, following `buildChallenges`'s filter/sort shape in
      `internal/web/challenges.go`; unit tests cover each filter alone,
      filters combined, and the empty-result case
- [x] 1.3 Add an empty-state path (`Found: false` or `len(Rows) == 0`)
      matching the dashboard's no-active-lab empty state; unit test
      asserts the view renders without a postmortem
- [x] 1.4 Add `GET /postmortems` to `internal/web/handlers.go`, wired to
      a new `postmortemsIndexHandler(cfg)`; test asserts a 200 and that
      the response contains no variant id or unrevealed hint text
- [x] 1.5 `go test ./internal/web/...` passes for this group in
      isolation

## 2. internal/web: sidebar link

- [x] 2.1 Add a "Postmortems" nav item to `templates/web/_sidebar.html`
      between Challenges and Topology, following the existing
      `sidebar-nav-item` markup pattern (placed right after Topology,
      still alongside the same four existing items — no ordering
      requirement in the spec)
- [x] 2.2 Extend `SidebarView.Current`'s documented value set in
      `internal/web/sidebar.go` to include `"postmortems"`, and pass it
      from the new handler; test asserts the nav item gets
      `sidebar-nav-current` only on the postmortems page

## 3. templates/web/postmortems.html

- [x] 3.1 Create `templates/web/postmortems.html` rendering
      `PostmortemsIndexView`: filter controls (stage, difficulty,
      status), a row list linking to each postmortem's existing detail
      page, and the empty state from 1.3
- [x] 3.2 Confirm via the existing "no `<script>` tag on any page"
      template test (or its equivalent) that the new page follows the
      same no-inline-script rule as every other page except the lab
      detail's live-tail script

## 4. internal/site: public postmortems index

- [x] 4.1 Add a `PostmortemsIndexView` (or extend the existing
      `IndexView`/row types) and a `BuildPostmortemsIndex` function to
      `internal/site/view.go`, built from the same catalog +
      progress.json + `published map[string]bool` inputs `BuildIndex`
      already takes, producing a flat list sorted by solved date, most
      recent first; unit tests cover a mix of published/unpublished
      postmortems and the zero-published case
- [x] 4.2 Add `templates/site/postmortems.html` reusing
      `templates/site/style.css`'s existing tokens
- [x] 4.3 Add a `writePostmortemsIndex` step to `internal/site/site.go`
      alongside the existing `writeIndex`/`writePostmortems`, writing to
      `site/postmortems/index.html`; wire it into `Build`. Unlike
      `writePostmortems`, this always creates `postmortems/` and writes
      `index.html` even when nothing is published (the index still
      renders, just empty) — required updating
      `TestBuildIgnoresPostmortemPathOutsideDir` in
      `site_security_test.go`, which had asserted the `postmortems/`
      directory never exists for a build with nothing publishable; that
      assertion is now replaced with the real invariant (no page or
      link for the specific out-of-dir postmortem, and the index still
      renders)
- [x] 4.4 Add a link to the new index from `site/index.html` (e.g. near
      the totals) so it's discoverable without knowing the URL
- [x] 4.5 Unit test: a postmortem with `publish: false` never appears
      in the built `site/postmortems/index.html` output (mirrors the
      existing "nothing leaks" test for `site/index.html`)
- [x] 4.6 Unit test: `site/postmortems/index.html`, once built, opens
      and renders fully from disk with no network connection (mirrors
      the existing offline test for `site/index.html`)
- [x] 4.7 `go test ./internal/site/...` passes

## 5. Verification

- [x] 5.1 `go test ./...` passes
- [x] 5.2 `go vet ./...` passes; `gofmt -l .` reports nothing for any
      file this change touches (a pre-existing, unrelated formatting
      issue in `internal/markdown/markdown.go`, an untracked file this
      change never edits, is out of scope here)
- [x] 5.3 Manual run, in an isolated copy of the repo (never the real
      `progress/progress.json`): built the binary, laid out an isolated
      temp fixture (two labs, three completed attempts, two postmortem
      files — one published, one draft), ran `breaklab serve`, and
      drove `/postmortems` with curl: default view lists both rows
      (most recent first), `?status=published` / `?status=draft`,
      `?stage=linux`, and `?difficulty=2` each filter correctly, and
      the lab whose *latest* attempt is a draft correctly supersedes its
      own earlier published postmortem for the same lab (per the
      documented per-lab dedup rule). Then ran `breaklab publish` from
      the same fixture: `site/postmortems/index.html` lists only the
      one lab whose latest attempt is published, and `site/index.html`
      links to it
- [x] 5.4 Manual check: curl'd `/postmortems`'s response body and
      confirmed no `<script>` tag and no unrevealed-hint/variant-id
      leakage beyond what the project's existing postmortem-slug
      convention already exposes post-solve (the slug embeds the
      variant name the same way every other postmortem link in this
      codebase already does — not new exposure introduced here)
