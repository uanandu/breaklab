## 1. Review-due computation

- [x] 1.1 Add `reviewInterval` constant and `ReviewDue(cat, pf) map[string]bool` to `internal/progress`, built alongside `LatestCompleteAttempts`, and verify with unit tests covering: never-completed lab, recently-completed lab, and a lab past the interval
- [x] 1.2 Add a helper that identifies the active session as a "review attempt" (lab already has a complete attempt before this one started) and verify with a unit test for both the review and non-review case

## 2. CLI surface

- [x] 2.1 Have `breaklab status` list review-due labs by id and verify with a CLI integration test asserting the listing appears for a lab seeded past the interval
- [x] 2.2 Have the `hint` command refuse to reveal a hint during a review attempt, with a clear message, and verify with a CLI integration test

## 3. Web surface

- [x] 3.1 Add `ReviewDue bool` to `LabRowView`, set it in `buildRoadmap` and `buildChallenges`, and verify with unit tests asserting it's true only for a lab past the interval
- [x] 3.2 Render a "Review due" badge on `roadmap.html` and `challenges.html` rows where `ReviewDue` is true, and verify by rendering the fixture catalog and asserting the badge markup appears
- [x] 3.3 Have the web UI's hint-reveal handler refuse a hint during a review attempt the same way the CLI does, and verify with an HTTP handler test asserting a non-200/refusal response — this came for free from the `runner.Hint` change (the web handler calls the same method) and is now covered by `TestLabHintRefusedDuringReviewAttempt`

## 4. End-to-end verification

- [x] 4.1 Run the full test suite (`go test ./...`) and confirm no regression in existing roadmap/challenge/status tests
- [x] 4.2 Manually start a review attempt on a lab with a seeded old `solved_at`, confirm no hint is offered, solve it, and confirm the due badge clears — covered by equivalent automated tests instead of a live Docker run, per explicit user confirmation
