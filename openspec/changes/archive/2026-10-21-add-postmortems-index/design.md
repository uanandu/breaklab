## Context

The local web UI already has a proven filter/sort pattern for exactly
this kind of "browse everything, narrow with query params" page:
`internal/web/challenges.go`'s `buildChallenges` combines a `filterLabs`
step (stage/difficulty/status/tag from `url.Values`) with a separate
`sortMatched` step, both operating on `*catalog.Catalog` +
`*progress.ProgressFile`. Postmortems are 1:1 with completed attempts
(`progress.Attempt.Postmortem`), so the data this new page needs —
lab/stage/difficulty from the catalog, solved date and postmortem path
from the attempt, publish status from the postmortem file's front
matter — is already loaded by existing code paths (`buildChallenges`'s
catalog walk, `internal/site.BuildIndex`'s `published map[string]bool`
lookup). See proposal.md - Why for the motivation; see
openspec/specs/web-ui/spec.md and openspec/specs/progress-site/spec.md
for the requirements this implements.

## Goals / Non-Goals

**Goals:**
- Reuse the existing filter/sort shape (`buildChallenges`) for the local
  `/postmortems` page rather than inventing a second pattern for
  "browse a list with query-param filters."
- Reuse the existing `BuildIndex` data-loading shape (catalog +
  progress.json + published-postmortem map) for the public index rather
  than adding a new site data source.
- Keep both pages read-only and additive: no new routes besides
  `GET /postmortems` locally, no new CLI flags, no schema changes.

**Non-Goals:**
- No full-text search across postmortem bodies (Challenges' `q` param
  searches lab titles/tags, not file contents) — filtering is by
  stage/difficulty/status only for this change. Full-text search over
  postmortem content can be a later, separate change if wanted.
- No pagination — the same "render the whole filtered list" approach
  Challenges uses today; the datasets (one postmortem per completed
  attempt) are small enough on a personal, single-user tool that this
  isn't a real scaling concern.
- No change to the postmortem detail view, its template, or its route
  on either surface.

## Decisions

**Local index is a new `PostmortemsIndexView` built by a new
`buildPostmortemsIndex`, not a mode flag on `buildChallenges`.**
Postmortems and labs are different units (a lab can have zero, one, or
multiple attempts/postmortems across resets), so bolting a
"postmortems mode" onto the lab-centric `ChallengesView` would force an
awkward 1:many reshape into a type built around 1 lab = 1 row.
Alternative considered: add a `ShowPostmortems bool` to
`ChallengesView` — rejected, it would couple two independently-evolving
views for a shallow code-reuse win; the actual reuse (the filter/sort
*shape*, not the type) is preserved by copying the pattern, not the
struct.

**Filter by status using the postmortem's own draft/published state,
not the attempt's.** An attempt is "complete" once its postmortem has
real content (per `internal/postmortem`'s validation); "published" is
the separate `publish: true` front-matter flag read the same way
`internal/site`'s `published map[string]bool` already does. The local
index's status filter is `draft` (complete, not published) vs.
`published`, reusing that existing boolean source rather than adding a
new one.

**Public index lives at `site/postmortems/index.html`, matching the
existing per-postmortem path shape `site/postmortems/<slug>.html`.**
Keeps every postmortem-related public URL under one `postmortems/`
prefix instead of a top-level `postmortems.html` sibling to `index.html`.
`internal/site.Build` gains one more write step alongside its existing
per-postmortem page loop, not a new top-level template category.

**Sidebar link, not a dashboard tile.** The sidebar already lists
Dashboard/Roadmap/Challenges/Topology; adding "Postmortems" there is
the minimal, consistent navigation change. No change to the dashboard's
existing "Recent" (5-item) list — it stays as a quick-glance preview,
now with an implicit "see all" available via the sidebar rather than a
widened cap.

## Risks / Trade-offs

- [Risk] A postmortem's `publish: true` flag can be set before the
  postmortem has real validated content (e.g., mid-edit). → Mitigation:
  the public index reuses `internal/site`'s existing `published`
  lookup, which is already computed from the same validation gate the
  per-postmortem page build uses — no new leakage surface is introduced
  by this change.
- [Risk] Reusing the filter/sort *pattern* without sharing code between
  `buildChallenges` and `buildPostmortemsIndex` risks the two drifting
  apart (e.g., one gains a sort option the other doesn't). → Mitigation:
  acceptable for this change — the two lists sort by different things
  (labs by difficulty/activity; postmortems by solved date/stage) and a
  shared generic sorter would be more abstraction than the two call
  sites currently justify. Revisit if a third such list appears.

## Open Questions

- Exact postmortem-index sort options (date solved, stage order,
  difficulty) are a presentation detail left to implementation — any
  reasonable default (solved date, most recent first, matching the
  dashboard's existing "Recent" order) satisfies the spec's scenarios
  as written.
