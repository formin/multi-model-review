---
description: Initialize config or show status for the multi-model-review workflow.
argument-hint: [init|status]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /multi-model-review:cross-review

Configure or inspect the cross-agent review workflow.

Arguments: `$ARGUMENTS`

## Behavior

No argument or `status`:

- Read `.cross-review/config.json`
- Report:
  - builder
  - reviewer
  - base ref
  - default package profile
  - most recent package directory and whether it was `micro`, `compact`, or `full`
- If config is missing, prompt the user to run `/multi-model-review:cross-review init`

`init`:

1. Ask which agent is the builder.
   - `claude-code`
   - `codex-cli`
   - `codex-mcp`
   - `codex-auto`
   - `gemini-cli`
   - custom ID

2. Ask which agent is the reviewer.
   - warn, but do not block, if builder and reviewer are the same
   - for Codex, explain:
     - `codex-auto`: default
     - `codex-cli`: long-running reviews
     - `codex-mcp`: short MCP checks only

3. Ask for base ref.
   - default `main`

4. Ask for default package profile.
   - `compact` (recommended)
   - `full`
   - `micro` (only appropriate for MCP smoke checks)
   - if reviewer is `codex-mcp`, force `micro`

5. Ask for the Spec Kit feature directory.
   - offer `specs/*/` choices when available
   - allow ad-hoc mode

6. Write `.cross-review/config.json`.

7. Add `.cross-review/` to `.gitignore` unless the user explicitly wants to commit it.

## Output style

Keep status output short and scannable.

## Refer to

- `/multi-model-review:review-package`
- `/multi-model-review:apply-review`
- `skills/multi-model-review/SKILL.md`
