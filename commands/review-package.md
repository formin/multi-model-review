---
description: Export a multi-model review package — bundles spec/plan/tasks/diff into a single markdown file for the reviewer model (Claude / Codex / Gemini / ...) to consume.
argument-hint: [spec-slug] [--base <ref>]
allowed-tools: [Read, Write, Glob, Grep, Bash(git diff:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git merge-base:*)]
---

# /review-package

Build a self-contained review-package file the reviewer model can run against, without any session context.

Arguments: `$ARGUMENTS`

## Steps

1. Load `.cross-review/config.json`. If missing, abort with: "Run `/cross-review init` first."

2. Resolve the spec-kit feature slug:
   - If `$ARGUMENTS` contains a positional slug, use `specs/<slug>/`.
   - Else pick the most recently modified `specs/*/` dir.
   - If no `specs/` dir exists, enter ad-hoc mode — describe the change purely from the diff.

3. Resolve the base ref:
   - If `--base <ref>` in arguments, use it.
   - Else use `base_ref` from config (default `main`).
   - Compute `BASE=$(git merge-base <ref> HEAD)`.

4. Gather inputs:
   - `specs/<slug>/spec.md`, `plan.md`, `tasks.md` (skip missing ones)
   - `git diff $BASE...HEAD`
   - `git log $BASE...HEAD --oneline`
   - Root `CLAUDE.md` + any `CLAUDE.md` under directories the diff touched

5. Select the reviewer template by file convention: `templates/<config.reviewer>-review-prompt.md`.
   - `codex-cli` → `templates/codex-review-prompt.md`
   - `claude-code` → `templates/claude-review-prompt.md`
   - `gemini-cli` → `templates/gemini-review-prompt.md`
   - Any other ID → `templates/<id>-review-prompt.md`
   - If the file doesn't exist, abort with: "No template for reviewer `<id>`. Drop one at `templates/<id>-review-prompt.md` — see existing templates for shape."

6. Render the template, substituting placeholders `{{SPEC}}`, `{{PLAN}}`, `{{TASKS}}`, `{{DIFF}}`, `{{LOG}}`, `{{CLAUDE_MD}}`, `{{BASE_REF}}`, `{{HEAD_REF}}`, `{{REPORT_PATH}}`.

7. Write to `.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/review-package.md` along with `metadata.json` (base, head, builder, reviewer, timestamp, slug).

8. Print to the user:
   - The package path
   - The exact command to run the reviewer. Examples:
     - Codex: `codex exec --file .cross-review/packages/<pkg>/review-package.md`
     - Claude: `claude -p "$(cat .cross-review/packages/<pkg>/review-package.md)"`
     - Gemini: `gemini --file .cross-review/packages/<pkg>/review-package.md`
   - Where the reviewer should save its report: `.cross-review/packages/<pkg>/review-report.md`

## Do not

- Do not run the reviewer model for the user.
- Do not include secrets from `.env`, `.envrc`, or similar — grep the diff for likely secrets (`AKIA`, `-----BEGIN`, `ghp_`, `sk-`) and refuse with a warning if found.
- Do not pipe content larger than ~200KB into the package — if the diff is huge, split by path and prompt the user to pick a subset.
