---
description: Cross-agent review workflow — init config, show status, or route to /review-package or /apply-review.
argument-hint: [init|status|package|apply]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /cross-review

Top-level entry point for the `spec-cross-review` plugin.

Arguments: `$ARGUMENTS`

## Behavior

Parse `$ARGUMENTS`:

- **empty** or `status` — read `.cross-review/config.json` and report the current builder/reviewer roles, base ref, and the most recent package (if any) with its state (awaiting-review / report-received / applied). If no config exists, prompt to run `/cross-review init`.
- `init` — interactively create `.cross-review/config.json`. Ask:
  1. Which agent is the builder? (claude-code | codex-cli)
  2. Which agent is the reviewer? (the other one — confirm, don't assume)
  3. Base ref to diff against (default: `main`)
  4. Spec-kit feature directory, if any (glob `specs/*/`; let user pick or skip)
  Write the config and add `.cross-review/` to `.gitignore` unless the user wants it committed.
- `package` — delegate to `/review-package` (same repo).
- `apply` — delegate to `/apply-review` (same repo).

## Output style

Short. One section per piece of information. No emojis. Link files as `.cross-review/config.json` etc. so the user can click through.

## Refer to

- Skill: `skills/cross-review/SKILL.md`
- Commands: `commands/review-package.md`, `commands/apply-review.md`
