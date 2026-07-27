---
description: Run equity research end to end from one input — a ticker, a holdings list, or a sector/theme
argument-hint: "[ticker | holdings list or file | sector/theme] [lang=en] [depth=N]"
---

Load the `auto-coverage` skill and run it against the input below.

Input: $ARGUMENTS

Follow the skill exactly:

1. Resolve today's actual date for the output folder — never write a placeholder.
2. Resolve the output language (default Traditional Chinese, 繁體中文; override with
   `lang=en` or an equivalent explicit directive) and the pipeline depth (default 5 — the
   full pipeline; override with `depth=N`, `through Task N`, `到第N輪`, or a target
   deliverable name). Strip these directives from the input before mode/venue/ticker
   parsing. Neither is inferred from a bare number.
3. Detect the mode from the remaining input shape (ticker / holdings / explore). Do not
   ask the user to pick a mode unless the input is genuinely ambiguous. Empty input means
   explore mode with the default universe.
4. State the detected mode, resolved venue, and resolved language/depth in one line, then
   start immediately.
5. Run the mode's task sequence, up to the resolved depth, **without pausing for
   confirmation between tasks** — this overrides the underlying skills' single-task and
   stop-for-review defaults. If depth < 5, stop cleanly after the target task and report
   it as a planned stop, not a failure.
6. Verify each task's prerequisite outputs exist, are non-empty, and are well-formed
   before starting the next task. On failure, stop and report which task failed and
   which file is missing or malformed.
7. Write every deliverable to its canonical path under `/Research/`, in the resolved
   language — file names, Excel tab names, and standard financial abbreviations stay in
   English regardless of language setting. Create no summary, roll-up, completion, or
   whole-folder-zip documents.

Examples:
- `/auto-coverage 2330.TW` — full pipeline, Traditional Chinese (default)
- `/auto-coverage 2330.TW depth=3` — stop after Task 3 (Valuation Analysis)
- `/auto-coverage TSM lang=en` — full pipeline, English
- `/auto-coverage 2408.TW 到第三輪就好` — stop after Task 3, Traditional Chinese
