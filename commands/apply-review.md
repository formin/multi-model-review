---
description: Ingest the reviewer's review-report.md and drive remediation.
argument-hint: [package-dir] [--min-confidence <0-100>]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash(git diff:*), Bash(git log:*)]
---

# /multi-model-review:apply-review

Consume a reviewer agent's `review-report.md` and walk through the findings.

Arguments: `$ARGUMENTS`

## Steps

1. Locate the package.
   - If `$ARGUMENTS` contains a path, use it.
   - Else pick the latest `.cross-review/packages/*/` that contains `review-report.md`.
   - If none exist, abort with: "No review-report.md found. Did the reviewer save its output?"

2. Parse `review-report.md` using `templates/review-report.md`.
   - `Context sufficiency`
   - `Verdict`
   - `Summary`
   - findings with:
     - `severity`
     - `confidence`
     - `location`
     - `summary`
     - `detail`
     - `suggested_fix`

3. Handle context sufficiency first.
   - If `Context sufficiency = needs-full-package`:
     - present the current findings
     - do not start broad edits yet
     - recommend `/multi-model-review:review-package --full`
     - if the findings are clearly path-local, recommend `/multi-model-review:review-package --paths <subset>`
     - stop unless the user explicitly wants to continue with the current report

4. Filter findings.
   - Default: drop findings with `confidence < 70`
   - `--min-confidence N` overrides
   - Group remaining findings by severity, then file

5. Present a numbered checklist.
   - Quote the reviewer's wording instead of paraphrasing it
   - Do not edit files yet

6. For each finding the user accepts:
   - read the target file at the cited line
   - propose one edit for that finding
   - apply only after confirmation
   - if two findings touch the same line, re-read between edits
   - record status in `.cross-review/packages/<pkg>/review-state.json`

7. After the pass:
   - summarize applied vs skipped findings
   - remind the user that a re-review means:
     - `/multi-model-review:review-package`
     - run the reviewer again

## Guardrails

- Never auto-apply a `critical` finding without explicit confirmation.
- Never merge multiple findings into one edit.
- If the compact package was declared insufficient, prefer re-packaging before broad remediation.
