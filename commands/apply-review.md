---
description: Ingest a review-report.md produced by the reviewer agent and walk through applying its findings.
argument-hint: [package-dir] [--min-confidence <0-100>]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash(git diff:*), Bash(git log:*)]
---

# /apply-review

Consume a reviewer agent's `review-report.md` and drive remediation.

Arguments: `$ARGUMENTS`

## Steps

1. Locate the package:
   - If `$ARGUMENTS` has a path, use it.
   - Else pick the latest `.cross-review/packages/*/` with a `review-report.md` present.
   - If none found, abort with: "No review-report.md found. Did the reviewer agent save its output?"

2. Parse `review-report.md` per `templates/review-report.md` schema. Each finding has:
   - `id` (stable across re-runs)
   - `severity` (critical | major | minor | info)
   - `confidence` (0–100)
   - `location` (file:line or file)
   - `summary`
   - `detail`
   - `suggested_fix` (optional)

3. Filter:
   - Default drop `confidence < 70`.
   - `--min-confidence N` overrides.
   - Group remaining findings by severity, then by file.

4. Present the findings as a numbered checklist. Do NOT edit any files yet.

5. For each finding the user accepts (interactively or via `apply all`):
   - Read the target file at the cited line.
   - Propose an Edit that implements the suggested fix, or — if the fix is non-obvious — ask the user how to proceed.
   - Apply only after user confirms (or in batch mode, apply all high-confidence ones but still pause for low-confidence).
   - Mark the finding `status: applied` in a sidecar `.cross-review/packages/<pkg>/review-state.json`.

6. After the pass:
   - Summarize what was applied vs skipped.
   - Remind the user: a re-review would be `/review-package` again, then run the other agent.

## Guardrails

- Preserve the reviewer's language when quoting findings. Do not paraphrase into stronger or weaker claims.
- Never auto-apply a finding flagged `severity: critical` without an explicit confirm, even in batch mode.
- Never merge multiple findings into a single edit; one Edit per finding so the state file stays coherent.
- If two findings target the same line, apply them sequentially and re-read between edits.
