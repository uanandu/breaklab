## Context

`progress/progress.json` already records `hints_used` per attempt and `status`; `internal/catalog.Lab.Tags` already carries each lab's topic tags. `internal/web`'s existing training-velocity aggregates (`buildTrainingVelocity`) already compute similar per-attempt statistics from the same two sources, so this follows an established pattern in the codebase rather than introducing a new one — see proposal.md - Impact.

## Goals / Non-Goals

**Goals:**
- Surface a real, already-recorded signal (hints needed per topic) without inventing new tracked data.
- Keep false positives low by requiring both a minimum sample size and a meaningful margin above the user's own baseline — a single bad lab must not label a whole tag a "pattern."

**Non-Goals:**
- No causal explanation of *why* a tag is hard — this reports the correlation, not a diagnosis.
- No LLM or free-text analysis of postmortem content — see proposal.md.

## Decisions

**Fixed thresholds (3 samples, 1.5x average), not configurable.** Alternative considered: make both numbers user-configurable — rejected as premature; there's no evidence yet that the defaults are wrong, and configurability for a single-user local tool with no one else's workload to tune for is complexity with no payoff until proven otherwise.

**Hints-used is the signal, not time-to-solve.** Hints-used is a cleaner proxy for "needed help" than elapsed time, which is confounded by interruptions, distractions, and labs that are simply longer. Alternative considered: use elapsed time per attempt (already recorded) — rejected as noisier for this specific purpose; nothing stops a later change from adding it as a second signal.

**Computed on read, not cached.** Same reasoning as `add-lab-retention-review`'s design: this is cheap enough (one pass over `progress.json`, already loaded for every status/dashboard render) to not need a persisted cache that could go stale.

## Risks / Trade-offs

[Small attempt counts make an average noisy even above the 3-sample floor] → Accepted for v1; the floor and multiplier are a blunt but honest filter, and this is advisory information the user can judge for themselves, not a blocking gate.

[A tag with inherently harder labs (by design) will always look like a "pattern" even with normal performance] → Acceptable; the point is prompting the user to look again at a topic, not asserting a flaw — a hard topic being flagged is not a false signal, it's arguably the correct one.

## Migration Plan

No data migration — reads only fields `progress.json` already has. Ships with no flag; the status/dashboard sections simply stay empty until a user's own history produces a qualifying pattern.
