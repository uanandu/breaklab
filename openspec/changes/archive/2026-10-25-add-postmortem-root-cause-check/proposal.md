## Why

A postmortem passes today purely on word count — each required section (Summary, Detection, Root cause, Fix, Prevention) just needs 20+ words of the user's own text. A postmortem can be verbose and entirely wrong: a user can write a plausible-sounding but incorrect root cause and it passes exactly as well as the correct one. The one check that would catch "you fixed the symptom but misdiagnosed why" never runs.

## What Changes

- `breaklab postmortem --validate` runs a new heuristic check against the Root cause section in addition to the existing word-count check: it must share at least one meaningful word with the solved attempt's actual variant's hint text (lab.yaml already carries `hints:` per variant — no new authoring field, no new lab.yaml schema).
- A Root cause section that shares no such word with the variant's hints gets flagged as a warning, not a hard failure — the check is a heuristic nudge ("this doesn't look like it mentions what was actually broken"), not a semantic grader, and a correct-but-differently-worded root cause must never be blocked from completing.
- The CLI prints which hint-derived words it expected to see when it flags a mismatch, so the warning is actionable rather than cryptic.
- Explicitly out of scope: true natural-language understanding or LLM-based grading of correctness; blocking completion on this check (it stays advisory); retroactively re-checking postmortems written before this change.

## Capabilities

### Modified Capabilities
- `postmortems`: adds an advisory root-cause-consistency check alongside the existing required-sections word-count check; does not change what makes a postmortem "complete" (still the existing word-count requirement).

## Impact

- `internal/postmortem`: new function comparing the Root cause section's words against the solved attempt's variant's `Hints` text.
- `internal/catalog`: no schema change — reuses the existing `Variant.Hints` field.
- `cmd/breaklab`: `postmortem --validate` output gains a warning line when the check doesn't find an overlap.
