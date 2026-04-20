---
description: Export a multi-model review package — bundles spec/plan/tasks/diff into a single markdown file for the reviewer model.
argument-hint: [slug] [--base <ref>]
allowed-tools: [Read, Write, Glob, Grep, Bash(git diff:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git merge-base:*)]
---

# /multi-model-review:review-package

Build a self-contained review-package file the reviewer model can run against, without any session context.

Arguments: `$ARGUMENTS`

## Steps

1. Load `.cross-review/config.json`. If missing, abort with: "Run `/multi-model-review:cross-review init` first."

2. Resolve the spec-kit feature slug:
   - If `$ARGUMENTS` contains a positional slug, use `specs/<slug>/`.
   - Else pick the most recently modified `specs/*/` directory.
   - If no `specs/` directory exists, enter ad-hoc mode — describe the change purely from the diff.

3. Resolve the base ref:
   - If `--base <ref>` is in arguments, use it.
   - Else use `base_ref` from config (default `main`).
   - Compute `BASE=$(git merge-base <ref> HEAD)`.

4. Gather inputs:
   - `specs/<slug>/spec.md`, `plan.md`, `tasks.md` (skip missing ones).
   - `git diff $BASE...HEAD`.
   - `git log $BASE...HEAD --oneline`.
   - Root `CLAUDE.md` + any `CLAUDE.md` under directories the diff touched.

5. Select the reviewer template by file convention: `templates/<config.reviewer>-review-prompt.md`.
   - `codex-auto` → `templates/codex-auto-review-prompt.md` (**default for Codex**; heuristic — try MCP first, fall back to CLI on `-32001` timeout)
   - `codex-cli` → `templates/codex-cli-review-prompt.md` (explicit CLI; for reviews expected to take minutes to tens of minutes)
   - `codex-mcp` → `templates/codex-mcp-review-prompt.md` (explicit MCP; only for <60s validations — the `mcp__codex__codex` tool hard-codes a `-32001 timed out` error at 60s)
   - `codex-cli` legacy alias → also resolves `templates/codex-review-prompt.md` if the `-cli-` variant is missing (backward compat with 0.1.0)
   - `claude-code` → `templates/claude-review-prompt.md`
   - `gemini-cli` → `templates/gemini-review-prompt.md`
   - Any other ID → `templates/<id>-review-prompt.md`
   - If the file doesn't exist, abort with: "No template for reviewer `<id>`. Drop one at `templates/<id>-review-prompt.md` — see existing templates for shape."

6. Render the template, substituting placeholders `{{SPEC}}`, `{{PLAN}}`, `{{TASKS}}`, `{{DIFF}}`, `{{LOG}}`, `{{CLAUDE_MD}}`, `{{BASE_REF}}`, `{{HEAD_REF}}`, `{{REPORT_PATH}}`.

7. Write to `.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/review-package.md` along with `metadata.json` (base, head, builder, reviewer, timestamp, slug).

8. Print to the user:
   - The package path.
   - The exact command to run the reviewer, matching the configured reviewer ID:

     | Reviewer ID   | Invocation to print                                                                                                                  | Suitable for              | Known failure                         |
     |---------------|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------|---------------------------------------|
     | `codex-auto`  | Heuristic: if the diff is small and the reviewer is expected to finish in <60s, try the `mcp__codex__codex` tool; otherwise — or on `-32001 timed out` — fall back to the `codex-cli` invocation below. | When unsure (**default**) | — (auto-falls back)                   |
     | `codex-mcp`   | Call the `mcp__codex__codex` MCP tool with the package file contents as the prompt and wait for the result inline.                  | <60s short validations    | `-32001 timed out` at 60s (hardcoded) |
     | `codex-cli`   | `codex exec --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/log.txt 2>&1 &` then attach with the `Monitor` tool (or `tail -f`) until it finishes, then move `log.txt` → `review-report.md` (or redirect straight to `review-report.md`). | Minutes to tens of minutes | No inherent timeout                  |
     | `claude-code` | `claude -p "$(cat .cross-review/packages/<pkg>/review-package.md)" > .cross-review/packages/<pkg>/review-report.md`                  | Any length                | —                                     |
     | `gemini-cli`  | `gemini --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md`                       | Any length                | —                                     |

   - Where the reviewer should save its report: `.cross-review/packages/<pkg>/review-report.md`.
   - If the reviewer is `codex-mcp` and the package exceeds ~20 KB (rough heuristic for a <60s budget), warn the user that a `-32001 timed out` is likely and offer to switch to `codex-cli` or `codex-auto` for this run.

## Do not

- Do not run the reviewer model for the user — the whole point of this plugin is that the user invokes the other agent themselves.
- Do not include secrets from `.env`, `.envrc`, or similar — grep the diff for likely secrets (`AKIA`, `-----BEGIN`, `ghp_`, `sk-`) and refuse packaging with a warning if found.
- Do not pipe content larger than ~200 KB into a review package — if the diff is huge, split by path and prompt the user to pick a subset.
