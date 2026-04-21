---
description: Export a compact multi-model review package for the reviewer model.
argument-hint: [slug] [--base <ref>] [--full] [--paths <glob,...>]
allowed-tools: [Read, Write, Glob, Grep, Bash(git diff:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git diff --stat:*), Bash(git diff --name-only:*), Bash(git diff --name-status:*)]
---

# /multi-model-review:review-package

Build a self-contained `review-package.md` file for the reviewer model.

Arguments: `$ARGUMENTS`

## Default behavior

Default to a **compact** package. Use `--full` only when:

- the user explicitly asks for exhaustive context
- a previous review report said `Context sufficiency: needs-full-package`

## Steps

1. Load `.cross-review/config.json`.
   - If missing, abort with: "Run `/multi-model-review:cross-review init` first."

2. Resolve the feature slug.
   - If `$ARGUMENTS` contains a positional slug, use `specs/<slug>/`.
   - Else pick the most recently modified `specs/*/` directory.
   - If no `specs/` directory exists, enter ad-hoc mode and describe the change from the diff only.

3. Resolve the base ref.
   - If `--base <ref>` is present, use it.
   - Else use `base_ref` from config, default `main`.
   - Compute `BASE = git merge-base <ref> HEAD`.

4. Resolve the scope.
   - If `--paths <glob,...>` is present, restrict later summaries and excerpts to those changed paths.
   - Build a changed-file list with `git diff --name-status $BASE...HEAD`.
   - Build diff stats with `git diff --stat $BASE...HEAD`.
   - Mark low-signal paths that should usually be omitted from compact packages:
     - lockfiles
     - generated artifacts
     - minified bundles
     - snapshots
     - vendored or third-party code

5. Pick the package profile.
   - If reviewer is `codex-mcp`, force `micro`.
   - Else if `--full` is present, use `full`.
   - Else use `config.package_profile` when present.
   - Else default to `compact`.

6. Gather raw inputs.
   - `specs/<slug>/spec.md`, `plan.md`, `tasks.md` when present
   - `git diff $BASE...HEAD`
   - `git log $BASE...HEAD --oneline`
   - root `CLAUDE.md`
   - any `CLAUDE.md` files under touched directories

7. Reduce the package before rendering.
   - Use RTK-style reduction even if RTK itself is not installed.
   - If RTK is installed, prefer RTK-assisted summaries when helpful.

   Build these placeholders:

   - `{{PACKAGE_PROFILE}}`
   - `{{PACKAGE_SCOPE}}`
   - `{{CHANGED_FILE_COUNT}}`
   - `{{CHANGED_LINE_COUNT}}`
   - `{{SPEC_BRIEF}}`: objective, acceptance criteria, explicit constraints, non-goals
   - `{{PLAN_BRIEF}}`: architecture decisions, risky modules, migrations, integrations
   - `{{TASKS_BRIEF}}`: only tasks tied to touched paths, unresolved work, and review-sensitive checkpoints
   - `{{RULES_BRIEF}}`: only rules relevant to touched paths
   - `{{LOG_BRIEF}}`: trimmed commit trail
   - `{{DIFF_MANIFEST}}`: grouped file manifest with one-line purpose per file
   - `{{DIFF_EXCERPTS}}`: high-signal hunks only
   - `{{PACKAGE_NOTES}}`: omissions, truncations, and scope notes

   Appendix placeholders:

   - `{{SPEC_APPENDIX}}`
   - `{{PLAN_APPENDIX}}`
   - `{{TASKS_APPENDIX}}`
   - `{{RULES_APPENDIX}}`
   - `{{RAW_DIFF_APPENDIX}}`

   Appendix rules:

   - In `micro` and `compact`, appendices should be short omission notes.
   - In `full`, appendices may contain raw source material.

8. Select the reviewer template.
   - `codex-auto` -> `templates/codex-auto-review-prompt.md`
   - `codex-cli` -> `templates/codex-cli-review-prompt.md`
   - `codex-mcp` -> `templates/codex-mcp-review-prompt.md`
   - `codex-cli` legacy fallback -> `templates/codex-review-prompt.md`
   - `claude-code` -> `templates/claude-review-prompt.md`
   - `gemini-cli` -> `templates/gemini-review-prompt.md`
   - custom reviewer -> `templates/<id>-review-prompt.md`
   - If missing, abort with: "No template for reviewer `<id>`. Add `templates/<id>-review-prompt.md`."

9. Render the template.
   - Substitute all package metadata, brief, appendix, and path placeholders.
   - Also substitute `{{BASE_REF}}`, `{{HEAD_REF}}`, and `{{REPORT_PATH}}`.

10. Write outputs under `.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/`.
   - `review-package.md`
   - `metadata.json` with base, head, builder, reviewer, timestamp, slug, package profile, and truncation notes

11. Print the next step.
   - Show the package path.
   - Show the selected package profile.
   - Show the exact reviewer command.
   - Remind the user where to write `review-report.md`.
   - If the report later says `Context sufficiency: needs-full-package`, recommend:
     - `/multi-model-review:review-package --full`
     - or `/multi-model-review:review-package --paths <subset>`

## Reviewer command hints

| Reviewer ID | Invocation to print |
|-------------|---------------------|
| `codex-auto` | Try MCP for very small packages, otherwise fall back to `codex exec --file ...` |
| `codex-mcp` | Call the `mcp__codex__codex` tool inline |
| `codex-cli` | `codex exec --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md` |
| `claude-code` | `claude -p "$(cat .cross-review/packages/<pkg>/review-package.md)" > .cross-review/packages/<pkg>/review-report.md` |
| `gemini-cli` | `gemini --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md` |

## Do not

- Do not run the reviewer for the user.
- Do not include secrets from `.env`, `.envrc`, or similar.
- Do not ship a huge raw package by default when a compact package would do.
- Do not exceed roughly 200 KB of packaged content. Split by path if needed.
