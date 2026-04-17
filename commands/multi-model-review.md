---
description: Multi-model code review workflow — dispatches to cross-review (status/init), review-package (export bundle for reviewer model), or apply-review (ingest report).
argument-hint: --cross-review [init|status] | --review-package [slug] [--base <ref>] | --apply-review [package-dir] [--min-confidence <0-100>]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /multi-model-review

Single entry point for the `spec-cross-review` plugin. Parse `$ARGUMENTS` and dispatch to one of three subcommands based on the first flag.

Arguments: `$ARGUMENTS`

## Dispatch

### `--cross-review [init|status]` — config + status

No second arg or `status`:
- Read `.cross-review/config.json` and report: builder role, reviewer role, base ref, most recent package directory and its state (awaiting-review / report-received / applied).
- If `.cross-review/config.json` is absent, prompt the user to run `/multi-model-review --cross-review init`.

`init`:
- Interactively create `.cross-review/config.json`. Ask in order:
  1. Which agent is the **builder**? (`claude-code` | `codex-cli` | `gemini-cli` | custom ID)
  2. Which agent is the **reviewer**? Confirm it's different from the builder; warn (don't block) if identical.
  3. Base ref to diff against (default: `main`).
  4. Spec-kit feature directory — glob `specs/*/` and let the user pick one, or skip for ad-hoc mode.
- Write the config.
- Add `.cross-review/` to `.gitignore` unless the user opts in to committing it.

### `--review-package [slug] [--base <ref>]` — export handoff bundle

1. Load `.cross-review/config.json`. If missing, abort with: "Run `/multi-model-review --cross-review init` first."

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
   - `codex-cli` → `templates/codex-review-prompt.md`
   - `claude-code` → `templates/claude-review-prompt.md`
   - `gemini-cli` → `templates/gemini-review-prompt.md`
   - Any other ID → `templates/<id>-review-prompt.md`
   - If the file doesn't exist, abort with: "No template for reviewer `<id>`. Drop one at `templates/<id>-review-prompt.md` — see existing templates for shape."

6. Render the template, substituting placeholders `{{SPEC}}`, `{{PLAN}}`, `{{TASKS}}`, `{{DIFF}}`, `{{LOG}}`, `{{CLAUDE_MD}}`, `{{BASE_REF}}`, `{{HEAD_REF}}`, `{{REPORT_PATH}}`.

7. Write to `.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/review-package.md` along with `metadata.json` (base, head, builder, reviewer, timestamp, slug).

8. Print to the user:
   - The package path.
   - The exact command to run the reviewer. Examples:
     - Codex: `codex exec --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md`
     - Claude: `claude -p "$(cat .cross-review/packages/<pkg>/review-package.md)" > .cross-review/packages/<pkg>/review-report.md`
     - Gemini: `gemini --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md`
   - Where the reviewer should save its report: `.cross-review/packages/<pkg>/review-report.md`.

### `--apply-review [package-dir] [--min-confidence <0-100>]` — ingest review report

1. Locate the package:
   - If `$ARGUMENTS` has a path, use it.
   - Else pick the latest `.cross-review/packages/*/` with a `review-report.md` present.
   - If none found, abort with: "No review-report.md found. Did the reviewer agent save its output?"

2. Parse `review-report.md` per `templates/review-report.md` schema. Each finding has:
   - `id` (stable across re-runs)
   - `severity` (critical | major | minor | info)
   - `confidence` (0–100)
   - `location` (file:line or file)
   - `summary`
   - `detail`
   - `suggested_fix` (optional)

3. Filter:
   - Default drop `confidence < 70`.
   - `--min-confidence N` overrides.
   - Group remaining findings by severity, then by file.

4. Present the findings as a numbered checklist. Do NOT edit any files yet.

5. For each finding the user accepts (interactively or via `apply all`):
   - Read the target file at the cited line.
   - Propose an Edit that implements the suggested fix, or — if the fix is non-obvious — ask the user how to proceed.
   - Apply only after user confirms (or in batch mode, apply all high-confidence ones but still pause for low-confidence).
   - Mark the finding `status: applied` in a sidecar `.cross-review/packages/<pkg>/review-state.json`.

6. After the pass:
   - Summarize what was applied vs skipped.
   - Remind the user: a re-review is `/multi-model-review --review-package` again, then run the reviewer model.

## Default (no flag)

If `$ARGUMENTS` is empty, behave as `--cross-review status`.

## Output style

Short. One section per piece of information. No emojis. Link files as `.cross-review/config.json` etc. so the user can click through.

## Guardrails (apply to all subcommands)

- Do not run the reviewer model for the user — the whole point of this plugin is that the user invokes the other agent themselves.
- Do not include secrets from `.env`, `.envrc`, or similar — grep the diff for likely secrets (`AKIA`, `-----BEGIN`, `ghp_`, `sk-`) and refuse packaging with a warning if found.
- Do not pipe content larger than ~200 KB into a review package — if the diff is huge, split by path and prompt the user to pick a subset.
- Preserve the reviewer's language when quoting findings. Do not paraphrase into stronger or weaker claims.
- Never auto-apply a finding flagged `severity: critical` without an explicit confirm, even in batch mode.
- Never merge multiple findings into a single edit; one Edit per finding so the state file stays coherent.
- If two findings target the same line, apply them sequentially and re-read between edits.

## Refer to

- Skill: `skills/cross-review/SKILL.md`
- Templates: `templates/<reviewer>-review-prompt.md`, `templates/review-report.md`
