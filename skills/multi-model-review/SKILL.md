---
name: multi-model-review
description: Use this skill when the user wants a cross-model Spec Kit workflow, including selecting a model for development spec authoring, selecting a model for actual implementation, exporting a cross-model code review package, or invoking /multi-model-review:cross-review, /multi-model-review:spec-handoff, /multi-model-review:review-package, or /multi-model-review:apply-review. The skill creates portable markdown handoffs between spec-author, implementation, and reviewer models.
license: MIT
---

# Multi-Model Spec Review

Coordinate a "spec author model -> implementation model -> reviewer model" workflow across different LLMs on top of GitHub Spec Kit artifacts. The skill does not invoke the other model directly. It creates portable handoff packages, and later ingests the review report the user brings back.

## When to activate

Activate when the user:

- asks to review with another model
- asks to write development specs with Codex, Opus, Claude, or another model
- asks to choose or record the model used for actual implementation
- wants a second opinion from Codex, Gemini, Claude, or another CLI model
- mentions Spec Kit artifacts in a review context
- invokes `/multi-model-review:cross-review`, `/multi-model-review:spec-handoff`, `/multi-model-review:review-package`, or `/multi-model-review:apply-review`

## Mental model

```text
feature brief
     |
     v
spec-authoring-prompt.md
     |
 user runs the SPEC model
     |
spec.md   plan.md   tasks.md
   \         |        /
    \        |       /
 user runs the IMPLEMENTATION model
             |
            diff
             |
     +-------v--------+
     | review-package |
     +-------+--------+
             |
   user runs the REVIEWER model
             |
     +-------v-------+
     | review-report |
     +-------+-------+
             |
 implementation model ingests findings
```

The skill owns three directions:

1. **Spec handoff**: gather feature context into a prompt for the configured spec author model
2. **Review export**: gather spec and git context into a compact review package
3. **Review ingest**: parse the review report and drive remediation

## Directory convention

```text
.cross-review/
  config.json
  spec-handoffs/<timestamp>-<slug>/
    spec-authoring-prompt.md
    spec-output.md
    metadata.json
  packages/<timestamp>-<slug>/
    review-package.md
    review-report.md
    metadata.json
```

Recommended `config.json` keys:

- `builder`
- `spec_author_model`
- `spec_author_options`
- `spec_author_profile`
- `implementation_model`
- `implementation_options`
- `reviewer`
- `base_ref`
- `spec_dir`
- `package_profile`

Model routing defaults:

- `spec_author_model`: prefer `codex-5.5`.
- `spec_author_options`: store model-specific options as structured JSON.
- `spec_author_profile`: optional legacy/display summary derived from `spec_author_options`.
- For Codex `codex-5.5` or `gpt-5.5`, default `spec_author_options = { "intelligence": "very-high", "speed": "normal" }`.
- Codex option values:
  - `intelligence`: `low`, `medium`, `high`, `very-high`
  - `speed`: `normal`, `fast`
- For Claude `opus-4.7`, `claude-opus-4.7`, or `claude-opus-4.7-1m`, default `spec_author_options = { "context": "1M", "workload": "high" }`.
- Claude option values:
  - `context`: `standard`, `1M`
  - `workload`: `low`, `normal`, `high`
- Treat legacy `work = max` as `workload = high` when mapping to the Claude UI shown to the user.
- `implementation_model`: default to `claude-sonnet-4.6` because implementation usually consumes the largest token budget.
- `implementation_options`: default to `{ "workload": "high" }` for Claude implementation models.
- Keep `builder` as the execution surface, such as `claude-code`, and keep `implementation_model` as the actual model descriptor.
- Keep `reviewer` separate from both spec author and implementation model.

## Spec handoff flow

Use `/multi-model-review:spec-handoff` before implementation when the user wants another model to produce or refine Spec Kit artifacts.

### 1. Resolve model routing

- Read `spec_author_model`, `spec_author_options`, optional `spec_author_profile`, `implementation_model`, and `implementation_options` from `.cross-review/config.json`.
- If `spec_author_options` is missing, infer it from the selected model:
  - `codex-5.5` or `gpt-5.5` -> `{ "intelligence": "very-high", "speed": "normal" }`
  - `opus-4.7`, `claude-opus-4.7`, or `claude-opus-4.7-1m` -> `{ "context": "1M", "workload": "high" }`
- If only legacy `spec_author_profile` is present, preserve it and add a package note that options were inferred from the display profile when possible.
- If `implementation_options` is missing for a Claude implementation model, infer `{ "workload": "high" }`.
- If the user asks for Opus, use `opus-4.7` with 1M context / high workload unless the user gives different model options.
- Derive `spec_author_profile` from `spec_author_options` for human-readable output, for example `intelligence=very-high, speed=normal`.

### 2. Gather spec inputs

- User feature brief from the command arguments or conversation
- Existing `spec.md`, `plan.md`, and `tasks.md` when refining an existing Spec Kit feature
- Relevant project rules, README, and touched files when they affect constraints
- Any explicit non-goals, acceptance criteria, deadlines, or integration boundaries

### 3. Render the spec-authoring template

Use `templates/spec-authoring-prompt.md`.

The template must tell the spec author model to:

- produce or update `spec.md`, `plan.md`, and `tasks.md`
- optimize the artifacts for implementation by `implementation_model`
- create token-conscious task slices suitable for Claude Sonnet 4.6 when that is the implementation model
- avoid implementing code
- output file blocks with target paths

### 4. Write outputs

Write:

- `spec-authoring-prompt.md`
- `metadata.json` with spec author model, spec author options, spec author profile, implementation model, timestamp, slug, and source notes

Print the selected spec author model, implementation model, prompt path, and expected `spec-output.md` path.

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

- `SPEC_AUTHOR_MODEL`
- `SPEC_AUTHOR_OPTIONS`
- `SPEC_AUTHOR_PROFILE`
- `IMPLEMENTATION_MODEL`
- `IMPLEMENTATION_OPTIONS`
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
- `metadata.json` with base/head, roles, spec author model, spec author options, implementation model, implementation options, profile, and truncation notes

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
- `commands/spec-handoff.md`
- `commands/review-package.md`
- `commands/apply-review.md`
- `templates/spec-authoring-prompt.md`
- `templates/review-report.md`
