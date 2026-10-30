## Why

`progress/progress.json` already records hints used and failed checks for every attempt, and every lab already carries topic tags, but nothing aggregates that data across runs. A user can need 3 hints on every networking-tagged lab and never see that as a pattern — each attempt is reviewed in isolation, so the one signal that would actually change how someone debugs next time (which category of problem they consistently struggle with) is sitting in the data and never surfaced.

## What Changes

- A new aggregation over existing `progress.json` attempts and catalog tags: average hints-used per tag, computed only across complete attempts.
- Tags where the average is markedly above the user's own overall average (a fixed threshold, e.g. 1.5x) AND have at least a minimum sample size (e.g. 3 completed labs with that tag) are surfaced as a "pattern" — everything below the sample-size floor is silently excluded rather than reported on noisy data.
- `breaklab status` prints a short "patterns" section listing these tags, if any exist, below the existing stage-completion summary.
- The web dashboard gets a small "Patterns" panel listing the same tags.
- Explicitly out of scope: free-text analysis of postmortem content, clustering, root-cause correctness grading (that's `add-postmortem-root-cause-check`'s concern, and this change has no dependency on it), any cross-user comparison (this is a single-user local tool — see PRODUCT.md).

## Capabilities

### New Capabilities
- `debugging-pattern-insights`: computing and surfacing which topic tags a user consistently needs more hints on, from attempt history already on disk.

## Impact

- `internal/progress`: new aggregation function reading existing `Attempts` (hints_used, lab id) and the catalog's per-lab tags; no new persisted fields.
- `cmd/breaklab`: `status` subcommand gains a patterns section.
- `internal/web`: `dashboard.go`/`dashboard.html` gain a small patterns panel.
