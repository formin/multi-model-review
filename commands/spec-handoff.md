---
description: Export a spec-authoring handoff prompt for the configured development spec model, model-specific options, and optional subagent task routing.
argument-hint: "[slug|feature brief] [--spec-model <model[:axis]@axis>] [--heavy] [--plan] [--subagents auto|off] [--model <id>] [--model-option <key=value>] [--dev-model <model[:axis]@axis>] [--implementation-model <id>] [--implementation-option <key=value>]"
allowed-tools: [Read, Write, Glob, Grep, Bash(git status:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git diff --name-only:*), Bash(git diff --stat:*)]
---

# speckit.multi-model-review.spec-handoff

Spec Kit extension command: `/speckit.multi-model-review.spec-handoff`

Legacy Claude plugin command: `/multi-model-review:spec-handoff`

Build a self-contained `spec-authoring-prompt.md` file for the model that will create or refine Spec Kit development artifacts.

Arguments: `$ARGUMENTS`

## Default behavior

Default to the model routing in `.cross-review/config.json`.

If config is missing or incomplete, use these defaults and tell the user to run
`/multi-model-review:cross-review init` when they want persistent settings:

- `spec = codex-5.5:xhigh@normal`
- `spec_heavy = opus-4.7:1m@max`
- `dev = sonnet-4.6@high`
- `subagent_fast = haiku-4.5@normal` when subagents are enabled

The legacy fields are still supported:

- `spec_author_model`
- `spec_author_options`
- `spec_author_profile`
- `implementation_model`
- `implementation_options`

If `subagent_routing.mode = "auto"`, include the resolved subagent routing contract in the handoff so the spec author can produce task slices with route hints. If the user passes `--subagents off`, omit route hints for this handoff only.

## Model spec options

Preferred detailed options:

- `--spec-model <model[:axis]@axis>` overrides `model_defaults.spec` for this handoff.
- `--heavy` selects `model_defaults.spec_heavy` for this handoff.
- `--plan` keeps the same handoff command but emphasizes `plan.md` and `tasks.md` output for implementation handoff planning.
- `--subagents auto|off` overrides `config.subagent_routing.mode` for this handoff only.
- `--dev-model <model[:axis]@axis>` may be accepted as an alias for `--implementation-model` when the user gives the new detailed syntax.

Compatibility options:

- `--model <id>` still overrides the spec author model.
- `--model-option <key=value>` still merges into spec author options.
- `--implementation-model <id>` still overrides the implementation model.
- `--implementation-option <key=value>` still merges into implementation options.

Detailed model specs use this grammar:

```text
<model>[:<axis-a>][@<axis-b>]
```

Examples:

- `codex-5.5:xhigh@normal`
- `opus-4.7:1m@max`
- `sonnet-4.6@high`

Parsing rules:

- Preserve the original spec as `raw` in metadata.
- Codex axis A is reasoning or intelligence: `low`, `medium`, `high`, `xhigh`, `very-high`.
- Codex axis B is speed or service tier: `normal`, `fast`, `priority`.
- Claude axis A is context when present: `standard`, `1m`, `1M`.
- Claude axis B is workload: `low`, `normal`, `high`, `max`.
- Normalize `xhigh` to legacy `intelligence=very-high` while preserving `reasoning=xhigh`.
- Normalize `1m` to `1M`.
- Normalize `sonnet-4.6` to `claude-sonnet-4.6`.
- Do not silently upgrade or downgrade any axis.

If both `--spec-model` and `--heavy` are present, the explicit `--spec-model`
wins and the command should mention that `--heavy` was ignored.

## Steps

1. Load `.cross-review/config.json` when present.

2. Resolve model routing.
   - `--spec-model <spec>` overrides `config.model_defaults.spec`.
   - Else `--heavy` uses `config.model_defaults.spec_heavy`.
   - Else use `config.model_defaults.spec`.
   - If `model_defaults` is missing, derive a structured spec from legacy `spec_author_model` and `spec_author_options`.
   - If the selected model is `codex-5.5` or `gpt-5.5`, default spec options to `{ "intelligence": "very-high", "reasoning": "xhigh", "speed": "normal" }`.
   - If the selected model is `opus-4.7`, `claude-opus-4.7`, or `claude-opus-4.7-1m`, default spec options to `{ "context": "1M", "workload": "max" }`.
   - `--model <id>` is a legacy override. Apply it only when `--spec-model` is not present.
   - Apply each `--model-option <key=value>` override after defaults and config.
   - Derive `spec_author_profile` from final options for display.
   - `--dev-model <spec>` or `--implementation-model <id>` overrides `config.model_defaults.dev`.
   - Default implementation model is `sonnet-4.6@high`.
   - If the implementation model is a Claude model, default implementation options to `{ "workload": "high", "allow_silent_upgrade": false }`.
   - Apply each `--implementation-option <key=value>` override after defaults and config.
   - Never infer a stronger implementation model because the spec looks large. Ask for an explicit override instead.
   - Resolve subagent routing from `config.subagent_routing`.
   - If `--subagents auto|off` is present, apply it for this handoff only.
   - If subagent routing is enabled but missing, infer the default role map:
     - scout -> `mmr-context-scout`, `model_defaults.subagent_fast` or `haiku-4.5@normal`
     - worker -> `mmr-implementation-worker`, `model_defaults.dev`
     - heavy planner -> `mmr-heavy-planner`, `model_defaults.spec_heavy`
     - review checker -> `mmr-review-checker`, Claude-compatible `model_defaults.review` or `model_defaults.dev`
   - Keep subagent routing descriptive in the prompt; do not run subagents during spec handoff generation.

3. Resolve the feature slug or brief.
   - If the first positional value matches `specs/<slug>/` or a directory under `specs/`, treat it as the feature slug.
   - Otherwise treat the remaining positional text as the feature brief.
   - Preserve quoted user requirements verbatim when they are short.
   - If neither exists, ask the user for the feature brief before writing a prompt.

4. Gather context.
   - Existing `specs/<slug>/spec.md`, `plan.md`, and `tasks.md` when present
   - Root `README.md` when present
   - Root `CLAUDE.md` and relevant nested `CLAUDE.md` files when present
   - `git status --short`
   - `git log --oneline -n 20`
   - `git diff --name-only` and `git diff --stat` when local changes exist

5. Reduce context before rendering.
   - Keep project constraints, acceptance criteria, public API contracts, data model notes, and risk areas.
   - Omit unrelated history, generated files, lockfile noise, and raw diffs unless they are directly relevant to the spec.
   - Preserve explicit user requirements verbatim when they are short.

6. Render `templates/spec-authoring-prompt.md`.
   - Substitute:
     - `{{SPEC_AUTHOR_MODEL}}`
     - `{{SPEC_AUTHOR_OPTIONS}}`
     - `{{SPEC_AUTHOR_PROFILE}}`
     - `{{IMPLEMENTATION_MODEL}}`
     - `{{IMPLEMENTATION_OPTIONS}}`
     - `{{SUBAGENT_ROUTING}}`
     - `{{FEATURE_SLUG}}`
     - `{{FEATURE_BRIEF}}`
     - `{{PROJECT_CONTEXT}}`
     - `{{EXISTING_SPEC}}`
     - `{{EXISTING_PLAN}}`
     - `{{EXISTING_TASKS}}`
     - `{{OUTPUT_NOTES}}`
   - If `--plan` is present, make `{{OUTPUT_NOTES}}` emphasize:
     - non-goals
     - implementation risks
     - task IDs that can be implemented independently
     - assigning implementation tasks to the resolved dev model and resolved subagent route
     - no production code edits

7. Write outputs under `.cross-review/spec-handoffs/<YYYYMMDD-HHMM>-<slug>/`.
   - `spec-authoring-prompt.md`
   - `metadata.json` with timestamp, slug, handoff type (`spec` or `plan`), selected detailed spec model, selected detailed implementation model, resolved subagent routing, legacy spec author model/options/profile, legacy implementation model/options, original arguments, and source notes

8. Print the next step.
   - Show the prompt path.
   - Show the selected spec model raw string and structured options.
   - Show the derived profile string.
   - Show the selected implementation model raw string and structured options.
   - Show whether subagent routing is `auto` or `off`; when enabled, print the role-to-agent/model table.
   - Remind the user to write the model output to `spec-output.md`.
   - Print a command hint only when the local CLI is obvious:
     - `codex-5.5`: `codex exec -m codex-5.5 - < .cross-review/spec-handoffs/<pkg>/spec-authoring-prompt.md > .cross-review/spec-handoffs/<pkg>/spec-output.md`
     - `opus-4.7`: `claude --model opus-4.7 -p "$(cat .cross-review/spec-handoffs/<pkg>/spec-authoring-prompt.md)" > .cross-review/spec-handoffs/<pkg>/spec-output.md`

## Do not

- Do not implement code.
- Do not run the spec author model for the user.
- Do not overwrite existing `spec.md`, `plan.md`, or `tasks.md` automatically.
- Do not include secrets from `.env`, `.envrc`, or similar files.
