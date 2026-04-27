---
description: Initialize config or show status for the multi-model-review workflow, including spec-author and implementation model routing.
argument-hint: [init|status]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /multi-model-review:cross-review

Configure or inspect the cross-agent Spec Kit workflow.

Arguments: `$ARGUMENTS`

## Behavior

No argument or `status`:

- Read `.cross-review/config.json`
- Report:
  - builder
  - spec author model
  - spec author options
  - spec author profile, when present or derived
  - implementation model
  - implementation options
  - reviewer
  - base ref
  - default package profile
  - most recent package directory and whether it was `micro`, `compact`, or `full`
- If config is missing, prompt the user to run `/multi-model-review:cross-review init`
  - If config exists but does not contain model routing keys, report them as missing and recommend rerunning `init` or adding:
  - `spec_author_model`
  - `spec_author_options`
  - `implementation_model`
  - `implementation_options`

`init`:

1. Ask which model should author or refine the development spec documents.
   - recommended: `codex-5.5` or `gpt-5.5`
     - store `spec_author_options = { "intelligence": "very-high", "speed": "normal" }`
   - alternative: `opus-4.7` or `claude-opus-4.7-1m`
     - store `spec_author_options = { "context": "1M", "workload": "high" }`
   - custom ID

2. Ask whether to use the default options for that model.
   - for Codex `codex-5.5` or `gpt-5.5`, default options are:
     - `intelligence = very-high`
     - `speed = normal`
   - for `opus-4.7`, default options are:
     - `context = 1M`
     - `workload = high`
   - accepted Codex values:
     - `intelligence = low | medium | high | very-high`
     - `speed = normal | fast`
   - accepted Claude values:
     - `context = standard | 1M`
     - `workload = low | normal | high`
   - for custom IDs, accept key/value pairs and store them under `spec_author_options`
   - derive `spec_author_profile` from the options for display, but treat `spec_author_options` as the source of truth

3. Ask which agent is the implementation builder surface.
   - `claude-code`
   - `codex-cli`
   - `codex-mcp`
   - `codex-auto`
   - `gemini-cli`
   - custom ID

4. Ask which model should do the actual development implementation.
   - default `claude-sonnet-4.6`
   - Claude model choices:
     - `claude-opus-4.7`
     - `claude-opus-4.7-1m`
     - `claude-sonnet-4.6`
     - `claude-haiku-4.5`
   - explain that this is separate from the builder surface and is optimized for token-heavy implementation work
   - custom ID

5. Ask which options should be used for the implementation model.
   - default for Claude implementation models:
     - `workload = high`
   - if the model is `claude-opus-4.7-1m`, also store `context = 1M`
   - accept key/value pairs and store them under `implementation_options`

6. Ask which agent is the reviewer.
   - warn, but do not block, if builder and reviewer are the same
   - for Codex, explain:
     - `codex-auto`: default
     - `codex-cli`: long-running reviews
     - `codex-mcp`: short MCP checks only

7. Ask for base ref.
   - default `main`

8. Ask for default package profile.
   - `compact` (recommended)
   - `full`
   - `micro` (only appropriate for MCP smoke checks)
   - if reviewer is `codex-mcp`, force `micro`

9. Ask for the Spec Kit feature directory.
   - offer `specs/*/` choices when available
   - allow ad-hoc mode

10. Write `.cross-review/config.json`.

11. Add `.cross-review/` to `.gitignore` unless the user explicitly wants to commit it.

Example config:

```json
{
  "builder": "claude-code",
  "spec_author_model": "codex-5.5",
  "spec_author_options": {
    "intelligence": "very-high",
    "speed": "normal"
  },
  "spec_author_profile": "intelligence=very-high, speed=normal",
  "implementation_model": "claude-sonnet-4.6",
  "implementation_options": {
    "workload": "high"
  },
  "reviewer": "codex-auto",
  "base_ref": "main",
  "spec_dir": "specs/001-auth-rework",
  "package_profile": "compact"
}
```

## Output style

Keep status output short and scannable.

## Refer to

- `/multi-model-review:review-package`
- `/multi-model-review:spec-handoff`
- `/multi-model-review:apply-review`
- `skills/multi-model-review/SKILL.md`
