## Why

Postmortems are currently only reachable one at a time, by direct link
from somewhere else. The local dashboard caps its "Recent" list to the
last 5 (`internal/web/dashboard.go`), and the only other way to open one
locally is to already know its slug (`GET /postmortems/{slug}`) — there
is no page that lists all of them. The public site has the same gap:
`BuildIndex` nests completed labs inside each stage's progress card
(`internal/site/view.go`), so a visitor has to scroll every stage to
find a published write-up; there is no flat list. As the number of
completed labs grows past a handful, both the learner and any public
visitor lose the ability to browse their own write-up history. The
Challenge browser already solved this exact problem for labs (search,
sort, filter by stage/difficulty/status/tag); postmortems have no
equivalent.

## What Changes

- Add `GET /postmortems` to the local web UI: a browsable list of every
  postmortem the user has written, draft and published alike, filterable
  by stage, difficulty, and status (draft/published), reusing the same
  filter/sort pattern already implemented for `/challenges`
  (`internal/web/challenges.go`). Each row links to the existing
  `GET /postmortems/{slug}` detail view, which is unchanged.
- Add a link to the new `/postmortems` page from the shared sidebar
  (`templates/web/_sidebar.html`), alongside the existing
  Dashboard/Roadmap/Challenges/Topology links.
- Add a public postmortems index page to the static site build
  (`breaklab publish`): a flat, sortable list of only the postmortems
  that are `publish: true`, each linking to its existing per-postmortem
  page under `site/postmortems/`. Built from the same data `BuildIndex`
  already loads (catalog, progress.json, published-postmortem set) —
  no new input sources.
- No changes to any existing route, page, or the postmortem detail view
  itself — this is purely additive.

## Capabilities

### New Capabilities

(none — this extends existing capabilities, it does not introduce a new one)

### Modified Capabilities

- `web-ui`: adds a new "Postmortems index view" requirement — a
  `GET /postmortems` route listing all of the user's postmortems,
  filterable by stage/difficulty/status, linked from the sidebar.
- `progress-site`: extends the "Static site build" requirement — adds a
  flat, sortable index of published postmortems to the generated
  `site/`, built from the same publish-flag rule already governing what
  leaves the machine (nothing with `publish: false` appears).

## Impact

- `internal/web/postmortem.go` (+tests): new `buildPostmortemsIndex`
  view builder and `PostmortemsIndexView`/row types, following the
  `buildChallenges` filter pattern.
- `internal/web/handlers.go`: register `GET /postmortems`.
- `templates/web/postmortems.html` (new), `templates/web/_sidebar.html`:
  new page template and sidebar link.
- `internal/site/view.go` (+tests): new `BuildPostmortemsIndex` (or an
  extension of `BuildIndex`) producing a flat, sorted list of published
  postmortems.
- `internal/site/site.go`: write the new page's output file during
  `breaklab publish`.
- `templates/site/postmortems.html` (new): public index template,
  reusing the existing `templates/site/style.css` tokens.
- No changes to `progress/progress.json`'s schema, any CLI command, or
  the existing postmortem detail routes/templates on either surface.
