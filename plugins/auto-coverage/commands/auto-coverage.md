---
description: Run equity research end to end from one input — a ticker, a holdings list, or a sector/theme
argument-hint: "[ticker | holdings list or file | sector/theme]"
---

Load the `auto-coverage` skill and run it against the input below.

Input: $ARGUMENTS

Follow the skill exactly:

1. Resolve today's actual date for the output folder — never write a placeholder.
2. Detect the mode from the input shape (ticker / holdings / explore). Do not ask the
   user to pick a mode unless the input is genuinely ambiguous. Empty input means
   explore mode with the default universe.
3. State the detected mode in one line, then start immediately.
4. Run the mode's full task sequence **without pausing for confirmation between tasks** —
   this overrides the underlying skills' single-task and stop-for-review defaults.
5. Verify each task's prerequisite outputs exist, are non-empty, and are well-formed
   before starting the next task. On failure, stop and report which task failed and
   which file is missing or malformed.
6. Write every deliverable to its canonical path under `/Research/`. Create no summary,
   roll-up, or completion documents.
