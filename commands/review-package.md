---
description: Export a compact multi-model review package for the reviewer model, including spec-author, implementation, subagent routing, and review model metadata.
argument-hint: "[slug|task-id] [--base <ref>] [--full] [--micro] [--paths <glob,...>] [--review-model <model[:axis]@axis>]"
allowed-tools: [Read, Write, Glob, Grep, Bash(git diff:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git diff --stat:*), Bash(git diff --name-only:*), Bash(git diff --name-status:*)]
---

# speckit.multi-model-review.review-package

Spec Kit extension command: `/speckit.multi-model-review.review-package`

Legacy Claude plugin command: `/multi-model-review:review-package`

Build a self-contained `review-package.md` file for the reviewer model.

Arguments: `$ARGUMENTS`

## Default behavior

Default to a **compact** package. Use `--full` only when:

- the user explicitly asks for exhaustive context
- a previous review report said `Context sufficiency: needs-full-package`

Use `--review-model <model[:axis]@axis>` to override the review model for a
single package.

Examples:

```text
/multi-model-review:review-package C-12 --review-model codex-5.5:high@normal
/multi-model-review:review-package L-30 --review-model codex-5.5:xhigh@normal
/multi-model-review:review-package L-50 --review-model codex-5.5:high@priority
```

## Model spec options

Detailed model specs use this grammar:

```text
<model>[:<axis-a>][@<axis-b>]
```

Parsing rules:

- Preserve the original spec as `raw` in metadata.
- Codex axis A is reasoning or intelligence: `low`, `medium`, `high`, `xhigh`, `very-high`.
- Codex axis B is speed or service tier: `normal`, `fast`, `priority`.
- Claude axis A is context when present: `standard`, `1m`, `1M`.
- Claude axis B is workload: `low`, `normal`, `high`, `max`.
- Normalize `xhigh` to legacy `intelligence=very-high` while preserving `reasoning=xhigh`.
- Normalize `1m` to `1M`.
- Do not silently upgrade or downgrade any axis.

Review model guidance:

- `codex-5.5:high@normal`: default cross-review pass, cost-aware.
- `codex-5.5:xhigh@normal`: use for non-trivial design reviews.
- `codex-5.5:high@priority`: use only when review turnaround is the bottleneck.

## Steps

1. Load `.cross-review/config.json`.
   - If missing, abort with: "Run `/multi-model-review:cross-review init` first."
   - Read `model_defaults`, `spec_author_model`, `spec_author_options`, optional `spec_author_profile`, `implementation_model`, `implementation_options`, optional `review_model`, optional `review_options`, and optional `subagent_routing`.
   - If model routing keys are missing, use:
     - `spec = codex-5.5:xhigh@normal`
     - `spec_heavy = opus-4.7:1m@max`
     - `dev = sonnet-4.6@high`
     - `review = codex-5.5:high@normal`
     - `subagent_fast = haiku-4.5@normal` when subagent routing is enabled
   - If the selected spec author model is `opus-4.7`, `claude-opus-4.7`, or `claude-opus-4.7-1m` and options are missing, use `spec_author_options = { "context": "1M", "workload": "max" }`.
   - If the selected implementation model is a Claude model and options are missing, use `implementation_options = { "workload": "high", "allow_silent_upgrade": false }`.
   - Derive `spec_author_profile` from `spec_author_options` when the profile string is missing.
   - If `subagent_routing` is missing, infer `mode = off` and add a package note that no subagent routing was recorded.
   - Include a package note that defaults were inferred from missing config.

2. Resolve the review model.
   - If `--review-model <spec>` is present, parse and use it for this package only.
   - Else use `config.model_defaults.review`.
   - Else derive a review model from legacy `review_model` and `review_options`.
   - Else default to `codex-5.5:high@normal`.
   - Derive `review_profile` from the final review options.

3. Resolve the feature slug or task/change ID.
   - If `$ARGUMENTS` contains a positional slug matching `specs/<slug>/` or `specs/*`, use `specs/<slug>/`.
   - If the positional value looks like `L-12`, `C-12`, or another task/change ID, keep it as `review_scope_id` and include matching task text when found.
   - Else pick the most recently modified `specs/*/` directory.
   - If no `specs/` directory exists, enter ad-hoc mode and describe the change from the diff only.

4. Resolve the base ref.
   - If `--base <ref>` is present, use it.
   - Else use `base_ref` from config, default `main`.
   - Compute `BASE = git merge-base <ref> HEAD`.

5. Resolve the scope.
   - If `--paths <glob,...>` is present, restrict later summaries and excerpts to those changed paths.
   - Build a changed-file list with `git diff --name-status $BASE...HEAD`.
   - Build diff stats with `git diff --stat $BASE...HEAD`.
   - Mark low-signal paths that should usually be omitted from compact packages:
     - lockfiles
     - generated artifacts
     - minified bundles
     - snapshots
     - vendored or third-party code

6. Pick the package profile.
   - If reviewer is `codex-mcp`, force `micro`.
   - Else if `--micro` is present, use `micro`.
   - Else if `--full` is present, use `full`.
   - Else use `config.package_profile` when present.
   - Else default to `compact`.

7. Gather raw inputs.
   - `specs/<slug>/spec.md`, `plan.md`, `tasks.md` when present
   - task text matching `review_scope_id` when present
   - `git diff $BASE...HEAD`
   - `git log $BASE...HEAD --oneline`
   - root `CLAUDE.md`
   - any `CLAUDE.md` files under touched directories

8. Reduce the package before rendering.
   - Use RTK-style reduction even if RTK itself is not installed.
   - If RTK is installed, prefer RTK-assisted summaries when helpful.

   Build these placeholders:

   - `{{PACKAGE_PROFILE}}`
   - `{{PACKAGE_SCOPE}}`
   - `{{SPEC_AUTHOR_MODEL}}`
   - `{{SPEC_AUTHOR_OPTIONS}}`
   - `{{SPEC_AUTHOR_PROFILE}}`
   - `{{IMPLEMENTATION_MODEL}}`
   - `{{IMPLEMENTATION_OPTIONS}}`
   - `{{SUBAGENT_ROUTING}}`
   - `{{REVIEW_MODEL}}`
   - `{{REVIEW_OPTIONS}}`
   - `{{REVIEW_PROFILE}}`
   - `{{CHANGED_FILE_COUNT}}`
   - `{{CHANGED_LINE_COUNT}}`
   - `{{SPEC_BRIEF}}`: objective, acceptance criteria, explicit constraints, non-goals
   - `{{PLAN_BRIEF}}`: architecture decisions, risky modules, migrations, integrations
   - `{{TASKS_BRIEF}}`: only tasks tied to touched paths, unresolved work, and review-sensitive checkpoints
   - `{{RULES_BRIEF}}`: only rules relevant to touched paths
   - `{{LOG_BRIEF}}`: trimmed commit trail
   - `{{DIFF_MANIFEST}}`: grouped file manifest with one-line purpose per file
   - `{{DIFF_EXCERPTS}}`: high-signal hunks only
   - `{{PACKAGE_NOTES}}`: omissions, truncations, model routing, and scope notes

   Appendix placeholders:

   - `{{SPEC_APPENDIX}}`
   - `{{PLAN_APPENDIX}}`
   - `{{TASKS_APPENDIX}}`
   - `{{RULES_APPENDIX}}`
   - `{{RAW_DIFF_APPENDIX}}`

   Appendix rules:

   - In `micro` and `compact`, appendices should be short omission notes.
   - In `full`, appendices may contain raw source material.

9. Select the reviewer template.
   - `codex-auto` -> `templates/codex-auto-review-prompt.md`
   - `codex-cli` -> `templates/codex-cli-review-prompt.md`
   - `codex-mcp` -> `templates/codex-mcp-review-prompt.md`
   - `codex-cli` legacy fallback -> `templates/codex-review-prompt.md`
   - `claude-code` -> `templates/claude-review-prompt.md`
   - `gemini-cli` -> `templates/gemini-review-prompt.md`
   - custom reviewer -> `templates/<id>-review-prompt.md`
   - If missing, abort with: "No template for reviewer `<id>`. Add `templates/<id>-review-prompt.md`."

10. Render the template.
   - Substitute all package metadata, brief, appendix, and path placeholders.
   - Also substitute `{{BASE_REF}}`, `{{HEAD_REF}}`, `{{REPORT_PATH}}`, `{{SPEC_AUTHOR_MODEL}}`, `{{SPEC_AUTHOR_OPTIONS}}`, `{{SPEC_AUTHOR_PROFILE}}`, `{{IMPLEMENTATION_MODEL}}`, `{{IMPLEMENTATION_OPTIONS}}`, `{{SUBAGENT_ROUTING}}`, `{{REVIEW_MODEL}}`, `{{REVIEW_OPTIONS}}`, and `{{REVIEW_PROFILE}}`.

11. Write outputs under `.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/`.
   - `review-package.md`
   - `metadata.json` with base, head, builder, reviewer, selected detailed review model, spec author model/options/profile, implementation model/options, subagent routing, timestamp, slug, review scope ID, package profile, original arguments, and truncation notes

12. Print the next step.
   - Show the package path.
   - Show the selected package profile.
   - Show the selected review model raw string and structured options.
   - Show the exact reviewer command.
   - Remind the user where to write `review-report.md`.
   - If the report later says `Context sufficiency: needs-full-package`, recommend:
     - `/multi-model-review:review-package --full`
     - or `/multi-model-review:review-package --paths <subset>`

## Reviewer command hints

| Reviewer ID | Invocation to print |
|-------------|---------------------|
| `codex-auto` | Try MCP for very small packages, otherwise fall back to `codex exec -m <review_model> --file ...` |
| `codex-mcp` | Call the `mcp__codex__codex` tool inline for tiny packages only |
| `codex-cli` | `codex exec -m <review_model> --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md` |
| `claude-code` | `claude -p "$(cat .cross-review/packages/<pkg>/review-package.md)" > .cross-review/packages/<pkg>/review-report.md` |
| `gemini-cli` | `gemini --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md` |

If the selected review model uses `speed=priority` and the local CLI does not
expose a priority flag, preserve `speed=priority` in metadata and say the user
must use the local Codex priority mechanism. Do not silently change it to normal.

## Do not

- Do not run the reviewer for the user.
- Do not include secrets from `.env`, `.envrc`, or similar.
- Do not ship a huge raw package by default when a compact package would do.
- Do not exceed roughly 200 KB of packaged content. Split by path if needed.
