---
name: multi-model-review
description: Use this skill when the user wants a cross-model code review workflow on top of Spec Kit artifacts, or invokes /multi-model-review:cross-review, /multi-model-review:review-package, or /multi-model-review:apply-review. The skill exports a compact review package for another model, then ingests that model's review report.
version: 0.2.0
license: MIT
---

# Multi-Model Spec Review

Coordinate a "builder agent vs reviewer agent" workflow across different LLMs on top of GitHub Spec Kit artifacts. The skill does not invoke the other model directly. It creates a portable handoff package, and later ingests the review report the user brings back.

## When to activate

Activate when the user:

- asks to review with another model
- wants a second opinion from Codex, Gemini, Claude, or another CLI model
- mentions Spec Kit artifacts in a review context
- invokes `/multi-model-review:cross-review`, `/multi-model-review:review-package`, or `/multi-model-review:apply-review`

## Mental model

```text
spec.md   plan.md   tasks.md   diff
   \         |        /        /
    \        |       /        /
     +-------v------+--------+
     | review-package.md     |
     +-----------+-----------+
                 |
      user runs the OTHER model
                 |
     +-----------v-----------+
     | review-report.md      |
     +-----------+-----------+
                 |
      builder ingests findings
```

The skill owns two directions:

1. **Export**: gather spec and git context into a compact review package
2. **Ingest**: parse the review report and drive remediation

## Directory convention

```text
.cross-review/
  config.json
  packages/<timestamp>-<slug>/
    review-package.md
    review-report.md
    metadata.json
```

Recommended `config.json` keys:

- `builder`
- `reviewer`
- `base_ref`
- `spec_dir`
- `package_profile`

## Export flow

Build a self-contained markdown package. Default to **compact-first** packaging.

### 1. Resolve the feature

- If the user gave a slug, use `specs/<slug>/`
- Else pick the most recently modified `specs/*/`
- If no Spec Kit directory exists, fall back to ad-hoc diff-only mode

### 2. Resolve scope

- Base ref: config value or `main`
- Optional path filter from `--paths`
- Reviewer mode:
  - `codex-mcp` implies `micro`
  - `codex-auto`, `codex-cli`, `claude-code`, `gemini-cli` should default to `compact`
  - `--full` overrides to `full`

### 3. Gather raw inputs

- `spec.md`
- `plan.md`
- `tasks.md`
- `git diff <base>...HEAD`
- `git log <base>...HEAD --oneline`
- relevant `CLAUDE.md` files

### 4. Reduce the package before rendering

Use RTK-style reduction:

- **Filtering**: drop generated noise, lockfile churn, snapshots, vendored paths unless the feature depends on them
- **Grouping**: create a grouped diff manifest before raw hunks
- **Truncation**: cap logs, long task lists, repeated rule text, and low-signal hunks
- **Deduplication**: collapse repeated context

Create these placeholders:

- `SPEC_BRIEF`
- `PLAN_BRIEF`
- `TASKS_BRIEF`
- `RULES_BRIEF`
- `LOG_BRIEF`
- `DIFF_MANIFEST`
- `DIFF_EXCERPTS`
- `PACKAGE_NOTES`

Appendix placeholders:

- `SPEC_APPENDIX`
- `PLAN_APPENDIX`
- `TASKS_APPENDIX`
- `RULES_APPENDIX`
- `RAW_DIFF_APPENDIX`

Rules:

- In `compact` and `micro`, appendices should be short omission notes, not raw dumps.
- In `full`, appendices may include the raw source material.
- If RTK is installed, prefer RTK-assisted summaries when gathering shell output. If it is not installed, create equivalent summaries manually.

### 5. Render the reviewer template

Use `templates/<reviewer>-review-prompt.md`.

The template must tell the reviewer to:

- validate spec-to-code alignment
- find real correctness, security, and rule issues
- score confidence 0-100
- report `Context sufficiency`
- write `review-report.md` using `templates/review-report.md`

### 6. Write outputs

Write:

- `review-package.md`
- `metadata.json` with base/head, roles, profile, and truncation notes

### 7. Print the next command

Print the exact reviewer command and the output path for `review-report.md`.

If the report later says `Context sufficiency: needs-full-package`, tell the user to rerun:

- `/multi-model-review:review-package --full`
- or `/multi-model-review:review-package --paths <subset>`

## Ingest flow

Read the latest `review-report.md` unless the user points to a specific package directory.

Parse:

- `Context sufficiency`
- `Verdict`
- `Summary`
- `Findings`

Default handling:

- drop findings with confidence below 70
- group remaining findings by severity and file
- do not edit until the user accepts a finding

Special handling:

- if `Context sufficiency = needs-full-package`, stop before making broad edits and recommend a `--full` or `--paths` rerun
- never auto-apply a `critical` finding without explicit approval

## Important constraints

- Do not run the reviewer model for the user.
- Do not package secrets.
- Do not dump raw artifacts in compact mode just because they are available.
- Preserve the reviewer's language when quoting findings back to the user.

## Related files

- `commands/cross-review.md`
- `commands/review-package.md`
- `commands/apply-review.md`
- `templates/review-report.md`
