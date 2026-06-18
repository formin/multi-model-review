---
description: Export a compact multi-model review package for the reviewer model, including spec-author, implementation, subagent routing, optional Headroom-aware context compression, and review model metadata.
argument-hint: "[slug|task-id] [--base <ref>] [--full] [--micro] [--paths <glob,...>] [--review-model <model[:axis]@axis>] [--headroom auto|off|required]"
allowed-tools: [Read, Write, Glob, Grep, Bash(git diff:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git diff --stat:*), Bash(git diff --name-only:*), Bash(git diff --name-status:*), Bash(headroom:*), Bash(rtk gain:*), mcp__headroom__headroom_compress, mcp__headroom__headroom_retrieve, mcp__headroom__headroom_stats]
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

Use Headroom opportunistically when it is available. In `auto` mode, the command
still builds a valid package when Headroom is missing; in `required` mode, abort
if Headroom MCP tools are not callable in the current host. Headroom should
compress only the large raw inputs left after normal package reduction, and each
compressed block must include enough notes for the reviewer to request the
original or ask for a full package.

Examples:

```text
/multi-model-review:review-package C-12 --review-model codex-5.5:high@normal
/multi-model-review:review-package L-30 --review-model codex-5.5:xhigh@normal
/multi-model-review:review-package L-50 --review-model codex-5.5:high@priority
/multi-model-review:review-package --headroom auto
/multi-model-review:review-package --headroom off
/multi-model-review:review-package --headroom required
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
- For unknown/custom models, if the segment after `:` is not one of the known axis values for that provider, treat the whole string as the native model ID. This preserves local-provider model tags such as `<local-model>:<tag>`.
- Treat Codex `--oss`, `oss_provider`, `model_provider`, and `[model_providers.<id>]` as CLI runtime settings, not model-axis values. Preserve local model IDs, including native provider tags that contain `:`, in metadata and keep provider selection in the command hint or Codex config.

Review model guidance:

- `codex-5.5:high@normal`: default cross-review pass, cost-aware.
- `codex-5.5:xhigh@normal`: use for non-trivial design reviews.
- `codex-5.5:high@priority`: use only when review turnaround is the bottleneck.

## Steps

When the work-assist orchestration layer is available, use omo or lazycodex for fast LSP/AST code exploration while gathering and reducing the package, and run an `mmr-review-checker` preflight under omo or `/ulw` when helpful. Fall back to native gathering otherwise, and never pass secrets into their prompts. See `skills/multi-model-review/SKILL.md` -> "Work-assist orchestration".

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
   - Read optional `context_compression`. If it is missing, use `{ "provider": "headroom", "mode": "auto", "min_tokens": 2000 }`.
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

7. Resolve Headroom-aware context compression.
   - Parse `--headroom auto|off|required`. If absent, use `config.context_compression.mode`; if that is absent, use `auto`.
   - `off`: do not call Headroom. Use the manual RTK-style reducers only.
   - `auto`: use Headroom only when it is available and the input block exceeds the configured `min_tokens`.
   - `required`: abort if Headroom MCP compression is unavailable.
   - Prefer Headroom MCP tools when the host exposes `headroom_compress`, `headroom_retrieve`, and `headroom_stats`.
   - The Headroom CLI is useful for install/status checks such as `headroom --version` or `headroom mcp status`, but do not assume a direct `headroom compress` subcommand exists.
   - If only the CLI is present and no Headroom MCP tool is callable, mark package-time Headroom compression unavailable and use manual summaries.
   - Do not start `headroom proxy`, `headroom wrap`, or any long-running background process from this command.
   - Do not double-compress content that already contains a Headroom or CCR retrieval marker.
   - If the external reviewer will not share the same local Headroom MCP/proxy store, keep compressed blocks self-describing and include enough diff excerpts for a `limited-but-actionable` review. Treat CCR hashes as optional retrieval aids, not as the only copy of review-critical evidence.

8. Gather raw inputs.
   - `specs/<slug>/spec.md`, `plan.md`, `tasks.md` when present
   - task text matching `review_scope_id` when present
   - `git diff $BASE...HEAD`
   - `git log $BASE...HEAD --oneline`
   - root `CLAUDE.md`
   - any `CLAUDE.md` files under touched directories

9. Reduce the package before rendering.
   - Use RTK-style reduction even if RTK itself is not installed.
   - If RTK is installed, prefer RTK-assisted summaries when helpful.
   - Apply manual filtering, grouping, truncation, and deduplication before invoking Headroom.
   - If Headroom is enabled and available, use it for remaining large raw blocks such as verbose diffs, logs, JSON tool outputs, search results, code excerpts, or long prose rules.
   - Preserve structure-critical data before compression: file paths, line numbers, error messages, function/class names, public API signatures, migrations, security-sensitive rules, and explicit acceptance criteria.
   - For each Headroom-compressed block, record:
     - source label, such as `git-diff`, `rules`, `tasks`, or `log`
     - original token estimate
     - compressed token estimate when available
     - savings percentage when available
     - transforms reported by Headroom when available
     - CCR/retrieval hash when available
     - retrieval query hints, such as file path, symbol, task ID, or error string
   - If Headroom fails, add a package note and continue with manual summaries unless `--headroom required` was set.
   - If `headroom_stats` is available, include a short final stats note in metadata.

   Build these placeholders:

   - `{{PACKAGE_PROFILE}}`
   - `{{PACKAGE_SCOPE}}`
   - `{{CONTEXT_COMPRESSION}}`
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
   - `{{PACKAGE_NOTES}}`: omissions, truncations, model routing, scope notes, Headroom compression notes, and retrieval hints

   Appendix placeholders:

   - `{{SPEC_APPENDIX}}`
   - `{{PLAN_APPENDIX}}`
   - `{{TASKS_APPENDIX}}`
   - `{{RULES_APPENDIX}}`
   - `{{RAW_DIFF_APPENDIX}}`

   Appendix rules:

   - In `micro` and `compact`, appendices should be short omission notes.
   - In `full`, appendices may contain raw source material.

10. Select the reviewer template.
   - `codex-auto` -> `templates/codex-auto-review-prompt.md`
   - `codex-cli` -> `templates/codex-cli-review-prompt.md`
   - `codex-mcp` -> `templates/codex-mcp-review-prompt.md`
   - `codex-cli` legacy fallback -> `templates/codex-review-prompt.md`
   - `claude-code` -> `templates/claude-review-prompt.md`
   - `gemini-cli` -> `templates/gemini-review-prompt.md`
   - `hermes` -> `templates/hermes-review-prompt.md`
   - custom reviewer -> `templates/<id>-review-prompt.md`
   - If missing, abort with: "No template for reviewer `<id>`. Add `templates/<id>-review-prompt.md`."

11. Render the template.
   - Substitute all package metadata, brief, appendix, and path placeholders.
   - Also substitute `{{BASE_REF}}`, `{{HEAD_REF}}`, `{{REPORT_PATH}}`, `{{CONTEXT_COMPRESSION}}`, `{{SPEC_AUTHOR_MODEL}}`, `{{SPEC_AUTHOR_OPTIONS}}`, `{{SPEC_AUTHOR_PROFILE}}`, `{{IMPLEMENTATION_MODEL}}`, `{{IMPLEMENTATION_OPTIONS}}`, `{{SUBAGENT_ROUTING}}`, `{{REVIEW_MODEL}}`, `{{REVIEW_OPTIONS}}`, and `{{REVIEW_PROFILE}}`.

12. Write outputs under `.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/`.
   - `review-package.md`
   - `metadata.json` with base, head, builder, reviewer, selected detailed review model, spec author model/options/profile, implementation model/options, subagent routing, timestamp, slug, review scope ID, package profile, original arguments, context compression mode, Headroom availability, compressed block metadata, stats note, and truncation notes

13. Print the next step.
   - Show the package path.
   - Show the selected package profile.
   - Show the Headroom mode and whether compression was applied.
   - Show the selected review model raw string and structured options.
   - Show the exact reviewer command.
   - For `codex-cli` and `codex-auto`, also show the local OSS alternative when useful: `codex exec --oss -m <review_model> --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md`.
   - Remind the user where to write `review-report.md`.
   - If the report later says `Context sufficiency: needs-full-package`, recommend:
     - `/multi-model-review:review-package --full`
     - or `/multi-model-review:review-package --paths <subset>`

14. Finish with the Completion token report.
   - Append the `**Token, Headroom & RTK**` block from `skills/multi-model-review/SKILL.md` (section "Completion token report").
   - This flow has the most concrete numbers: sum the `compressed_blocks` `original_tokens`/`compressed_tokens` written to `metadata.json` for the Headroom line, and read `headroom_stats` for session totals when available.
   - Read `rtk gain` for the RTK line. Report the two layers separately from measured stats only; mark `used=no` with the next applicable command when a layer did not engage. Do not estimate or expose secrets.

   ```text
   **Token, Headroom & RTK**
   - **RTK**: used=<yes|no> — saved ≈ <N> tok (<P>%) · via `rtk gain`
   - **Headroom**: used=<yes|no> — saved ≈ <N> tok (<P>%) · via `headroom_stats` / package `compressed_blocks`
   - **Combined saved**: ≈ <RTK+Headroom> tok   (only when both layers are measured)
   - **Work-assist** (orchestration, usage only): ulw=<used|n/a>, omo=<used|n/a>, lazycodex=<used|n/a>, superpowers=<used|n/a>
   - **Subagent routing** (usage only): scout=<used|n/a>, worker=<used|n/a>, heavy-planner=<used|n/a>, review-checker=<used|n/a>
   ```

## Reviewer command hints

| Reviewer ID | Invocation to print |
|-------------|---------------------|
| `codex-auto` | Try MCP for very small packages, otherwise fall back to `codex exec -m <review_model> --file ...`; for local providers use `codex exec --oss -m <review_model> --file ...` |
| `codex-mcp` | Call the `mcp__codex__codex` tool inline for tiny packages only |
| `codex-cli` | `codex exec -m <review_model> --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md`; local: `codex exec --oss -m <review_model> --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md` |
| `claude-code` | `claude -p "$(cat .cross-review/packages/<pkg>/review-package.md)" > .cross-review/packages/<pkg>/review-report.md` |
| `gemini-cli` | `gemini --file .cross-review/packages/<pkg>/review-package.md > .cross-review/packages/<pkg>/review-report.md` |
| `hermes` | Open a Hermes Agent session, provide `.cross-review/packages/<pkg>/review-package.md`, and save the reply as `.cross-review/packages/<pkg>/review-report.md` |

If the selected review model uses `speed=priority` and the local CLI does not
expose a priority flag, preserve `speed=priority` in metadata and say the user
must use the local Codex priority mechanism. Do not silently change it to normal.

## Do not

- Do not run the reviewer for the user.
- Do not include secrets from `.env`, `.envrc`, or similar.
- Do not ship a huge raw package by default when a compact package would do.
- Do not exceed roughly 200 KB of packaged content. Split by path if needed.
- Do not rely on a transient CCR hash as the only source for evidence needed by an external reviewer who cannot access the same local Headroom store.
