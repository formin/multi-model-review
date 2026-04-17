---
description: Init config or show status for the multi-model-review workflow.
argument-hint: [init|status]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /multi-model-review:cross-review

Configure or inspect the cross-agent review workflow.

Arguments: `$ARGUMENTS`

## Behavior

No second arg or `status`:
- Read `.cross-review/config.json` and report: builder role, reviewer role, base ref, most recent package directory and its state (awaiting-review / report-received / applied).
- If `.cross-review/config.json` is absent, prompt the user to run `/multi-model-review:cross-review init`.

`init`:
- Interactively create `.cross-review/config.json`. Ask in order:
  1. Which agent is the **builder**? (`claude-code` | `codex-cli` | `gemini-cli` | custom ID)
  2. Which agent is the **reviewer**? Confirm it's different from the builder; warn (don't block) if identical.
  3. Base ref to diff against (default: `main`).
  4. Spec-kit feature directory — glob `specs/*/` and let the user pick one, or skip for ad-hoc mode.
- Write the config.
- Add `.cross-review/` to `.gitignore` unless the user opts in to committing it.

## Output style

Short. One section per piece of information. No emojis. Link files as `.cross-review/config.json` etc. so the user can click through.

## Refer to

- `/multi-model-review:review-package` — export the handoff bundle for the reviewer
- `/multi-model-review:apply-review` — ingest the reviewer's report
- Skill: `skills/multi-model-review/SKILL.md`
