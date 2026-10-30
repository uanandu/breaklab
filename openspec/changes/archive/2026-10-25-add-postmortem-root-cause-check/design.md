## Context

`internal/postmortem` already parses a postmortem's front matter (lab, variant) and its required sections for the word-count check. `internal/catalog.Variant.Hints` already holds author-written hint text for the variant that was actually applied — the one piece of ground truth about what was really broken that the system already has on disk, see proposal.md - Impact.

## Goals / Non-Goals

**Goals:**
- Catch the specific failure mode of "postmortem is well-formed but describes the wrong thing" without requiring new lab authoring work across ~160 existing labs.
- Keep the check strictly advisory — never block or regress an already-correct postmortem on a wording mismatch.

**Non-Goals:**
- No semantic or LLM-based correctness grading — see proposal.md.
- No new per-variant authoring field; the check must work with the hints every variant already has today.

## Decisions

**Reuse `Variant.Hints` as the ground-truth vocabulary, not a new field.** Alternative considered: add a dedicated `root_cause_keywords` field to each variant — rejected because it would require touching every one of ~160 existing `lab.yaml` files for a heuristic check that hints already give enough signal for (a hint necessarily describes the real breakage, since it's written to guide the user toward it).

**Word-overlap heuristic, not fuzzy/semantic matching.** Lowercase, strip punctuation, drop a small fixed stopword list and anything under 4 characters, then check for any overlap between the Root cause section's words and the variant's hints' words. Alternative considered: Levenshtein/fuzzy matching or an embedding-based similarity score — rejected as disproportionate complexity (and in the embedding case, a new dependency and possibly a network call) for a check whose whole value is being a cheap, explainable nudge, not a grader.

**Advisory, never blocking.** The check prints a warning but never changes the postmortem's pass/fail outcome or the attempt's completion status. Alternative considered: block completion on a mismatch — rejected because a true positive rate low enough to trust as a hard gate isn't achievable with word overlap alone; false positives (a correct but differently-worded root cause) would incorrectly block real completions, which is worse than the status quo.

## Risks / Trade-offs

[Word overlap can both false-positive (coincidental shared word) and false-negative (correct root cause, zero shared vocabulary)] → Mitigated by keeping the check advisory only; it's a nudge the user can disregard, not a gate they can be wrongly blocked by.

[Stopword list is a judgment call] → Keep it small and reviewed in code, not configurable; a wrong omission just means an occasional false flag, not a functional break, since the check never blocks anything.

## Migration Plan

No data migration. The check runs only at validate-time against postmortems not yet marked complete; existing completed postmortems are never re-checked (see proposal.md - explicitly out of scope).
