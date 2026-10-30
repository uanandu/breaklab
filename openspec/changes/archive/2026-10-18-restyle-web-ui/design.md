## Context

`templates/web/style.css` (local web UI) and `templates/site/style.css`
(public site, built by `breaklab publish`) are two independent,
hand-written stylesheets with no shared tokens — colors and spacing are
literal values repeated in each. Both are plain CSS with no build step,
loaded by `html/template` pages; the local UI additionally uses HTMX for
the few interactive bits (filters, live status). See proposal.md - Why
for the motivation.

## Goals / Non-Goals

**Goals:**
- One shared token set (CSS custom properties) driving both stylesheets.
- A deliberate "incident dashboard" visual direction: dark by default,
  monospace accents for ids/commands/ports (echoing the `TICKET.md` /
  `docker exec` text the product already centers), a small, purposeful
  accent palette for status and difficulty.
- Light mode via `prefers-color-scheme`, not a toggle — no JS state to
  add given the no-JS-build-step constraint.
- Status and difficulty always paired with text/shape, not color alone.

**Non-Goals:**
- No new routes, template data, or HTMX endpoints.
- No theme switcher UI or persisted preference — OS-level preference only.
- No change to the challenge browser's filtering logic (stage/difficulty/
  status/tag, from the topic-tags change) — only how its results render.
- No icon font or external asset — any glyphs used are plain Unicode/text,
  keeping `site/index.html` fully self-contained per the progress-site
  "Works offline" scenario.

## Decisions

**Dark-first terminal aesthetic, light mode via media query.**
BreakLab's whole premise is "you're debugging a broken machine" — the
ticket text, the `docker exec` command, the check output are all
terminal-flavored. A dark background with a monospace accent face for
those elements leans into that instead of fighting it. Alternative
considered: a conventional light SaaS-dashboard look — rejected as
generic and less tied to the product's actual voice. `prefers-color-scheme`
over a toggle: a toggle needs JS and somewhere to persist the choice
(localStorage, a cookie) for no real benefit to a single-user tool that
already runs with the visitor's own OS theme preference set once.

**CSS custom properties as the token layer, no preprocessor.**
A `:root { --bg: ...; --accent: ...; }` block (duplicated, or `@import`ed,
into both `templates/web/style.css` and `templates/site/style.css` since
`breaklab publish` copies `templates/site/` standalone and the two surfaces
are served from different roots) keeps the "no JS build step" constraint
intact. Alternative considered: a tiny Go text/template step that renders
both CSS files from one source at `go generate` time — rejected as
unnecessary complexity for ~40 lines of shared variables; plain
duplication of one small `:root` block is cheap enough to keep in sync by
hand, and `go vet`/CI don't need to care about CSS drift the way they do
about Go code.

**Status/difficulty as badge components, not cell coloring.**
Rendering `<span class="badge badge-complete">complete</span>` instead of
coloring the whole row or cell keeps the text label always present (the
accessibility requirement in the new spec) and is the smallest markup
change needed — existing `{{.Status}}` / `{{.Difficulty}}` template
expressions get wrapped, not restructured.

## Risks / Trade-offs

- [Two stylesheets duplicating one token block can drift] → keep the
  shared `:root` block small (a dozen or so variables) and identical
  text in both files; a follow-up task adds a cheap test asserting the
  two files' token blocks match byte-for-byte, so drift fails CI instead
  of surfacing as a visual bug.
- [Dark-first design increases contrast-tuning effort] → pick colors
  against WCAG AA contrast ratios for text/background and verify the
  handful of accent colors (status, difficulty) meet 3:1 against their
  background at minimum.
- [Markup changes to shared templates could regress the topic-tag filter
  or other recent functional changes] → restyling touches the same
  `templates/web/challenges.html` rows that carry tag chips; run the
  existing `internal/web` tests plus a manual pass of `breaklab serve`
  after the change, not just a visual diff.

## Migration Plan

No data migration. Rollout is: land the token + CSS changes, restyle each
template file, verify locally (`breaklab serve`, `breaklab publish` +
open `site/index.html` from disk), then ship. Rollback is a plain git
revert — no schema, no persisted state, no feature flag needed.
