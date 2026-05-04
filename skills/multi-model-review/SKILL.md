---
name: multi-model-review
description: Use this skill when the user wants a cross-model Spec Kit workflow, including selecting detailed model options for development spec authoring, heavy spec authoring, actual implementation, automatic subagent routing, and review, exporting a cross-model code review package, or invoking /speckit.multi-model-review.cross-review, /speckit.multi-model-review.spec-handoff, /speckit.multi-model-review.review-package, /speckit.multi-model-review.apply-review, or the legacy /multi-model-review:* commands. The skill creates portable markdown handoffs between spec-author, implementation, reviewer, and subagent-routed task slices.
license: MIT
---

# Multi-Model Spec Review

Coordinate a "spec author model -> implementation model -> reviewer model" workflow across different LLMs on top of GitHub Spec Kit artifacts. External models still move through portable handoff packages. Claude Code implementation work can additionally use the plugin's subagents so scout, planner, worker, and local review tasks run on situation-appropriate models.

## When to activate

Activate when the user:

- asks to review with another model
- asks to write development specs with Codex, Opus, Claude, or another model
- asks to choose or record the model used for actual implementation
- asks to split work across subagents or automatically choose models for delegated tasks
- wants a second opinion from Codex, Gemini, Claude, or another CLI model
- mentions Spec Kit artifacts in a review context
- gives detailed model specs such as `codex-5.5:xhigh@normal`, `opus-4.7:1m@max`, `sonnet-4.6@high`, or `codex-5.5:high@priority`
- invokes `/speckit.multi-model-review.cross-review`, `/speckit.multi-model-review.spec-handoff`, `/speckit.multi-model-review.review-package`, `/speckit.multi-model-review.apply-review`, or the legacy `/multi-model-review:*` commands

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
     optional subagent routing
   scout / planner / worker / checker
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

The skill owns four directions:

1. **Spec handoff**: gather feature context into a prompt for the configured spec author model
2. **Subagent routing**: record how implementation slices should choose scout, planner, worker, or local checker models
3. **Review export**: gather spec and git context into a compact review package
4. **Review ingest**: parse the review report and drive remediation

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
.claude/agents/
  mmr-*.md        # optional project-local overrides for plugin subagents
```

Recommended `config.json` keys:

- `builder`
- `model_defaults`
- `spec_author_model`
- `spec_author_options`
- `spec_author_profile`
- `spec_heavy_model`
- `spec_heavy_options`
- `spec_heavy_profile`
- `implementation_model`
- `implementation_options`
- `review_model`
- `review_options`
- `review_profile`
- `reviewer`
- `base_ref`
- `spec_dir`
- `package_profile`
- `subagent_routing`

Plugin-provided subagents:

- `mmr-context-scout`: fast read-only exploration, default Claude Code model `haiku`
- `mmr-implementation-worker`: scoped edits and tests, default Claude Code model `sonnet`
- `mmr-heavy-planner`: cross-cutting or ambiguous planning, default Claude Code model `opus`
- `mmr-review-checker`: local read-only preflight, default Claude Code model `sonnet`

## Detailed model specs

The existing `/multi-model-review:*` command surface is the only directive surface. Do not introduce another alias.

Accept detailed model specs in this grammar:

```text
<model>[:<axis-a>][@<axis-b>]
```

Examples:

- `codex-5.5:xhigh@normal`
- `codex-5.5:high@priority`
- `opus-4.7:1m@max`
- `sonnet-4.6@high`

Provider mappings:

- Codex:
  - model examples: `codex-5.5`, `gpt-5.5`
  - axis A: `low`, `medium`, `high`, `xhigh`, `very-high`
  - axis B: `normal`, `fast`, `priority`
- Claude:
  - model examples: `opus-4.7`, `sonnet-4.6`, `haiku-4.5`, `claude-*`
  - axis A: `standard`, `1m`, `1M` context when present
  - axis B: `low`, `normal`, `high`, `max` workload
- Gemini or custom:
  - preserve provider-specific axes without inventing mappings

Parsing rules:

- Preserve the original string as `raw`.
- Normalize `xhigh` to legacy display `intelligence=very-high`, but keep `reasoning=xhigh` in `model_defaults`.
- Normalize `1m` to `1M`.
- Normalize `sonnet-4.6` to `claude-sonnet-4.6`.
- Do not silently upgrade or downgrade any axis.
- Do not replace the selected implementation model with a stronger model unless the user explicitly requests it.

Model routing defaults:

- `model_defaults.spec`: `codex-5.5:xhigh@normal`.
- `model_defaults.spec_heavy`: `opus-4.7:1m@max`.
- `model_defaults.dev`: `sonnet-4.6@high` with `allow_silent_upgrade=false`.
- `model_defaults.review`: `codex-5.5:high@normal`.
- `model_defaults.subagent_fast`: `haiku-4.5@normal`.
- Keep legacy fields updated from `model_defaults` for older commands and templates.
- Keep `builder` as the execution surface, such as `claude-code`, and keep `implementation_model` as the actual model descriptor.
- Keep `reviewer` separate from both implementation model and `review_model`.

## Subagent auto-routing

Use subagent routing when the user wants implementation work divided into task slices or when `subagent_routing.mode = "auto"` in `.cross-review/config.json`.

Default role map:

- scout -> `mmr-context-scout`, model key `subagent_fast`
- worker -> `mmr-implementation-worker`, model key `dev`
- heavy planner -> `mmr-heavy-planner`, model key `spec_heavy`
- review checker -> `mmr-review-checker`, model key `review` only for Claude-compatible review models, otherwise `dev`

Selection rules:

- Choose `scout` for read-only codebase discovery, dependency mapping, and compact context summaries.
- Choose `heavy planner` for cross-cutting design, migrations, security-sensitive changes, unclear requirements, and large Spec Kit decomposition.
- Choose `worker` for scoped code and test edits after the task slice is clear.
- Choose `review checker` for local read-only preflight after an implementation slice or before exporting a review package.

When Claude Code can pass a per-invocation model to the Agent tool, use the resolved model for that subagent call. Otherwise, prefer project-local `.claude/agents/mmr-*.md` overrides generated by `/multi-model-review:cross-review init --subagents auto`; project agents override plugin agents.

Do not write Codex, Gemini, or unknown provider IDs into Claude Code subagent frontmatter unless the local Claude Code installation explicitly supports that custom model. Preserve those models in handoff metadata and use the existing CLI/package path.

## Spec handoff flow

Use `/multi-model-review:spec-handoff` before implementation when the user wants another model to produce or refine Spec Kit artifacts.

### 1. Resolve model routing

- Read `model_defaults`, `spec_author_model`, `spec_author_options`, optional `spec_author_profile`, `implementation_model`, and `implementation_options` from `.cross-review/config.json`.
- If the user passes `--spec-model <model[:axis]@axis>`, use it for this handoff only.
- If the user passes `--heavy`, use `model_defaults.spec_heavy`.
- If the user passes `--plan`, keep using `/multi-model-review:spec-handoff` but emphasize `plan.md` and `tasks.md` output for implementation handoff planning.
- If both `--spec-model` and `--heavy` are present, the explicit `--spec-model` wins and you should say so.
- If `model_defaults` is missing, infer it from legacy fields:
  - `codex-5.5` or `gpt-5.5` -> `{ "intelligence": "very-high", "reasoning": "xhigh", "speed": "normal" }`
  - `opus-4.7`, `claude-opus-4.7`, or `claude-opus-4.7-1m` -> `{ "context": "1M", "workload": "max" }`
- If only legacy `spec_author_profile` is present, preserve it and add a package note that options were inferred from the display profile when possible.
- If `implementation_options` is missing for a Claude implementation model, infer `{ "workload": "high", "allow_silent_upgrade": false }`.
- If the user asks for Opus heavy mode, use `opus-4.7:1m@max` unless the user gives different model options.
- Derive `spec_author_profile` from `spec_author_options` for human-readable output, for example `intelligence=very-high, reasoning=xhigh, speed=normal`.
- Resolve `subagent_routing` from config. If absent, treat it as `off` unless the user explicitly asks for subagent splitting.
- If subagent routing is enabled, include the role map and selection rules in `SUBAGENT_ROUTING`; do not invoke subagents while creating the spec handoff.

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
- add route hints to `tasks.md` such as `[route:scout]`, `[route:heavy-planner]`, `[route:worker]`, or `[route:review-checker]` when subagent routing is enabled
- avoid implementing code
- output file blocks with target paths

### 4. Write outputs

Write:

- `spec-authoring-prompt.md`
- `metadata.json` with the selected detailed spec model, spec author model/options/profile, selected detailed implementation model, implementation options, resolved subagent routing, timestamp, slug, original arguments, and source notes

Print the selected spec author model, implementation model, subagent routing mode, prompt path, and expected `spec-output.md` path.

## Export flow

Build a self-contained markdown package. Default to **compact-first** packaging.

### 1. Resolve the feature

- If the user gave a slug, use `specs/<slug>/`
- Else pick the most recently modified `specs/*/`
- If no Spec Kit directory exists, fall back to ad-hoc diff-only mode

### 2. Resolve scope

- Base ref: config value or `main`
- Optional path filter from `--paths`
- Optional `--review-model <model[:axis]@axis>` override for a single review package
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
- `SUBAGENT_ROUTING`
- `REVIEW_MODEL`
- `REVIEW_OPTIONS`
- `REVIEW_PROFILE`
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
- `metadata.json` with base/head, roles, selected detailed review model, spec author model, spec author options, implementation model, implementation options, subagent routing, profile, and truncation notes

### 7. Print the next command

Print the exact reviewer command and the output path for `review-report.md`. For Codex review models, include the selected model ID in the command hint and preserve `speed=priority` in metadata even if the local CLI requires a separate priority mechanism.

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
- Do not silently upgrade the configured implementation model. If `model_defaults.dev` says `sonnet-4.6@high`, keep that routing unless the user explicitly overrides it.
- Automatic subagent routing may choose a different configured role model for a different kind of task, but record the selected role, agent, model key, and reason in metadata or the task plan.
- Subagents should not spawn subagents. Keep orchestration in the main conversation or slash command.

## Related files

- `commands/cross-review.md`
- `commands/spec-handoff.md`
- `commands/review-package.md`
- `commands/apply-review.md`
- `templates/spec-authoring-prompt.md`
- `templates/review-report.md`
- `agents/mmr-context-scout.md`
- `agents/mmr-implementation-worker.md`
- `agents/mmr-heavy-planner.md`
- `agents/mmr-review-checker.md`
