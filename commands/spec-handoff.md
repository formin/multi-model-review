---
description: Export a spec-authoring handoff prompt for the configured development spec model and model-specific options.
argument-hint: [slug|feature brief] [--model <id>] [--model-option <key=value>] [--implementation-model <id>] [--implementation-option <key=value>]
allowed-tools: [Read, Write, Glob, Grep, Bash(git status:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git diff --name-only:*), Bash(git diff --stat:*)]
---

# /multi-model-review:spec-handoff

Build a self-contained `spec-authoring-prompt.md` file for the model that will create or refine Spec Kit development artifacts.

Arguments: `$ARGUMENTS`

## Default behavior

Default to the model routing in `.cross-review/config.json`.

If config is missing or incomplete, use these defaults and tell the user to run `/multi-model-review:cross-review init` when they want persistent settings:

- `spec_author_model = codex-5.5`
- `spec_author_options = { "intelligence": "very-high", "speed": "normal" }`
- `implementation_model = claude-sonnet-4.6`
- `implementation_options = { "workload": "high" }`

If `$ARGUMENTS` includes `--model opus-4.7`, `--model claude-opus-4.7`, or `--model claude-opus-4.7-1m`, use:

- `spec_author_model = <selected Claude Opus model>`
- `spec_author_options = { "context": "1M", "workload": "high" }`

If `$ARGUMENTS` includes `--implementation-model <id>`, use that implementation model for this handoff only.

If `$ARGUMENTS` includes one or more `--model-option <key=value>` values, merge them into `spec_author_options` after applying the selected model defaults.

If `$ARGUMENTS` includes one or more `--implementation-option <key=value>` values, merge them into `implementation_options` after applying the selected implementation model defaults.

## Steps

1. Load `.cross-review/config.json` when present.

2. Resolve model routing.
   - `--model <id>` overrides `config.spec_author_model`.
   - If the selected model is `codex-5.5` or `gpt-5.5`, default `spec_author_options` to `{ "intelligence": "very-high", "speed": "normal" }`.
   - If the selected model is `opus-4.7`, `claude-opus-4.7`, or `claude-opus-4.7-1m`, default `spec_author_options` to `{ "context": "1M", "workload": "high" }`.
   - If config already contains `spec_author_options`, use it unless `--model` changes the model.
   - If config only contains legacy `spec_author_profile`, derive key/value options from comma-separated `key=value` pairs when possible.
   - Convert legacy `work=max` to `workload=high` for Claude options unless the user explicitly keeps `work`.
   - Apply each `--model-option <key=value>` override after defaults and config.
   - Derive `spec_author_profile` from final `spec_author_options` for display.
   - `--implementation-model <id>` overrides `config.implementation_model`.
   - Default implementation model is `claude-sonnet-4.6`.
   - If the implementation model is a Claude model, default `implementation_options` to `{ "workload": "high" }`.
   - If the implementation model is `claude-opus-4.7-1m`, also set `context = 1M`.
   - Apply each `--implementation-option <key=value>` override after defaults and config.

3. Resolve the feature slug or brief.
   - If the first positional value matches `specs/<slug>/` or a directory under `specs/`, treat it as the feature slug.
   - Otherwise treat the remaining positional text as the feature brief.
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
     - `{{FEATURE_SLUG}}`
     - `{{FEATURE_BRIEF}}`
     - `{{PROJECT_CONTEXT}}`
     - `{{EXISTING_SPEC}}`
     - `{{EXISTING_PLAN}}`
     - `{{EXISTING_TASKS}}`
     - `{{OUTPUT_NOTES}}`

7. Write outputs under `.cross-review/spec-handoffs/<YYYYMMDD-HHMM>-<slug>/`.
   - `spec-authoring-prompt.md`
   - `metadata.json` with timestamp, slug, spec author model, spec author options, spec author profile, implementation model, implementation options, and source notes

8. Print the next step.
   - Show the prompt path.
   - Show the selected spec author model and options.
   - Show the derived profile string.
   - Show the selected implementation model.
   - Show the selected implementation options.
   - Remind the user to write the model output to `spec-output.md`.
   - Print a command hint only when the local CLI is obvious:
     - `codex-5.5`: `codex exec -m codex-5.5 - < .cross-review/spec-handoffs/<pkg>/spec-authoring-prompt.md > .cross-review/spec-handoffs/<pkg>/spec-output.md`
     - `opus-4.7`: `claude --model opus-4.7 -p "$(cat .cross-review/spec-handoffs/<pkg>/spec-authoring-prompt.md)" > .cross-review/spec-handoffs/<pkg>/spec-output.md`

## Do not

- Do not implement code.
- Do not run the spec author model for the user.
- Do not overwrite existing `spec.md`, `plan.md`, or `tasks.md` automatically.
- Do not include secrets from `.env`, `.envrc`, or similar files.
