## 1. Aggregation

- [x] 1.1 Add a per-tag hints-used aggregation function to `internal/progress` reading `Attempts` and the catalog's tags, verified with unit tests covering: single qualifying tag, tag below sample-size floor, tag below the multiplier threshold
- [x] 1.2 Add the pattern-threshold filter (>=3 complete attempts, >=1.5x overall average) as a pure function over the aggregation's output, verified with a unit test table covering boundary values (exactly 3 samples, exactly 1.5x) — implemented as one `Patterns` function rather than two separate ones (aggregation + filter), since nothing else in the codebase needs the pre-filter aggregation on its own; boundary values are covered by `TestPatternsExactlyAtSampleFloorQualifies`/`TestPatternsExactlyAtMultiplierThresholdQualifies`

## 2. CLI surface

- [x] 2.1 Print a patterns section in `breaklab status` when at least one tag qualifies, verified with a CLI integration test using a seeded progress.json that produces one qualifying tag
- [x] 2.2 Verify `breaklab status` prints no patterns section when no tag qualifies, via a CLI integration test using a seeded progress.json with no attempts past the floor

## 3. Web surface

- [x] 3.1 Add a "Patterns" panel to the dashboard view, populated from the same aggregation, verified with a unit test on `buildDashboard` asserting the panel's rows match a fixture's expected qualifying tags
- [x] 3.2 Verify the panel renders nothing when there are no qualifying tags, via a template rendering test

## 4. End-to-end verification

- [x] 4.1 Run the full test suite (`go test ./...`) and confirm no regression in existing status/dashboard tests
- [x] 4.2 Manually seed a progress.json with a tag's labs needing consistently more hints and confirm it appears in both `breaklab status` and the dashboard — verified live in a scratch directory: `breaklab status` printed "Patterns: networking avg 6.0 hints over 3 labs", and the real dashboard.html served it in a "Patterns" panel
