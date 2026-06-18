---
name: multi-model-review
description: Use this skill when the user wants a cross-model Spec Kit workflow, including selecting detailed model options for development spec authoring, heavy spec authoring, actual implementation, automatic subagent routing, Headroom-aware context compression, and review, exporting a cross-model code review package, or invoking /speckit.multi-model-review.cross-review, /speckit.multi-model-review.spec-handoff, /speckit.multi-model-review.review-package, /speckit.multi-model-review.apply-review, or the legacy /multi-model-review:* commands. The skill creates portable markdown handoffs between spec-author, implementation, reviewer, and subagent-routed task slices. When available, it also drives execution through an optional work-assist orchestration layer (omo, UltraWork, lazycodex, Superpowers) and falls back to its own subagents when those tools are absent. When a flow completes, it ends with a measured "Token, Headroom & RTK" report that states the RTK and Headroom token savings separately and lists work-assist tool usage. Review state follows the Harness-1 (pat-jj/harness-1) state-externalizing pattern: curated evidence, evidence links, and verification records persist as recoverable files under .cross-review/.
license: MIT
---

# Multi-Model Spec Review

Coordinate a "spec author model -> implementation model -> reviewer model" workflow across different LLMs on top of GitHub Spec Kit artifacts. External models still move through portable handoff packages. Claude Code implementation work can additionally use the plugin's subagents so scout, planner, worker, and local review tasks run on situation-appropriate models.

## When to activate

Activate when the user:

- asks to review with another model
- asks to write development specs with Codex, Opus, Claude, or another model
- asks to choose or record the model used for actual implementation
- asks to split work across subagents or automatically choose models for delegated tasks
- wants a second opinion from Codex, Gemini, Claude, or another CLI model
- asks to use Codex OSS mode, Ollama, LM Studio, or another local Codex provider for spec authoring or review
- mentions Spec Kit artifacts in a review context
- gives detailed model specs such as `codex-5.5:xhigh@normal`, `opus-4.7:1m@max`, `sonnet-4.6@high`, or `codex-5.5:high@priority`
- invokes `/speckit.multi-model-review.cross-review`, `/speckit.multi-model-review.spec-handoff`, `/speckit.multi-model-review.review-package`, `/speckit.multi-model-review.apply-review`, or the legacy `/multi-model-review:*` commands

## Mental model

```text
feature brief
     |
     v
spec-authoring-prompt.md
     |
 user runs the SPEC model
     |
spec.md   plan.md   tasks.md
   \         |        /
    \        |       /
 user runs the IMPLEMENTATION model
             |
     optional subagent routing
   scout / planner / worker / checker
             |
            diff
             |
     +-------v--------+
     | review-package |
     +-------+--------+
             |
   user runs the REVIEWER model
             |
     +-------v-------+
     | review-report |
     +-------+-------+
             |
 implementation model ingests findings
```

The skill owns four directions:

1. **Spec handoff**: gather feature context into a prompt for the configured spec author model
2. **Subagent routing**: record how implementation slices should choose scout, planner, worker, or local checker models
3. **Review export**: gather spec and git context into a compact review package
4. **Review ingest**: parse the review report and drive remediation

## Directory convention

```text
.cross-review/
  config.json
  spec-handoffs/<timestamp>-<slug>/
    spec-authoring-prompt.md
    spec-output.md
    metadata.json
  packages/<timestamp>-<slug>/
    review-package.md
    review-report.md
    metadata.json
.claude/agents/
  mmr-*.md        # optional project-local overrides for plugin subagents
```

Recommended `config.json` keys:

- `builder`
- `model_defaults`
- `spec_author_model`
- `spec_author_options`
- `spec_author_profile`
- `spec_heavy_model`
- `spec_heavy_options`
- `spec_heavy_profile`
- `implementation_model`
- `implementation_options`
- `review_model`
- `review_options`
- `review_profile`
- `reviewer`
- `base_ref`
- `spec_dir`
- `package_profile`
- `subagent_routing`
- `context_compression`

Plugin-provided subagents:

- `mmr-context-scout`: fast read-only exploration, default Claude Code model `haiku`
- `mmr-implementation-worker`: scoped edits and tests, default Claude Code model `sonnet`
- `mmr-heavy-planner`: cross-cutting or ambiguous planning, default Claude Code model `opus`
- `mmr-review-checker`: local read-only preflight, default Claude Code model `sonnet`

## Detailed model specs

The existing `/multi-model-review:*` command surface is the only directive surface. Do not introduce another alias.

Accept detailed model specs in this grammar:

```text
<model>[:<axis-a>][@<axis-b>]
```

Examples:

- `codex-5.5:xhigh@normal`
- `codex-5.5:high@priority`
- `opus-4.7:1m@max`
- `sonnet-4.6@high`

Provider mappings:

- Codex:
  - model examples: `codex-5.5`, `gpt-5.5`
  - axis A: `low`, `medium`, `high`, `xhigh`, `very-high`
  - axis B: `normal`, `fast`, `priority`
- Claude:
  - model examples: `opus-4.7`, `sonnet-4.6`, `haiku-4.5`, `claude-*`
  - axis A: `standard`, `1m`, `1M` context when present
  - axis B: `low`, `normal`, `high`, `max` workload
- Gemini or custom:
  - preserve provider-specific axes without inventing mappings

Codex OSS/local provider handling:

- Treat Codex `--oss`, `oss_provider`, `model_provider`, and `[model_providers.<id>]` as Codex CLI runtime configuration, not as extra values in the model spec grammar.
- When the user asks for Ollama, LM Studio, or another local OSS Codex provider, keep the selected model string in handoff metadata and print command hints with `codex exec --oss -m <model>`.
- For custom Codex providers, assume the user has configured Codex `config.toml`; preserve the raw model and note that provider selection happens outside this extension.
- Do not write Codex provider IDs such as `openai`, `ollama`, `lmstudio`, or custom Codex provider IDs into Claude Code subagent frontmatter unless the local Claude Code installation explicitly supports them.

Parsing rules:

- Preserve the original string as `raw`.
- Normalize `xhigh` to legacy display `intelligence=very-high`, but keep `reasoning=xhigh` in `model_defaults`.
- Normalize `1m` to `1M`.
- Normalize `sonnet-4.6` to `claude-sonnet-4.6`.
- For unknown/custom models, if the segment after `:` is not one of the known axis values for that provider, treat the whole string as the native model ID. This preserves local-provider model tags such as `<local-model>:<tag>`.
- Do not silently upgrade or downgrade any axis.
- Do not replace the selected implementation model with a stronger model unless the user explicitly requests it.

Model routing defaults:

- `model_defaults.spec`: `codex-5.5:xhigh@normal`.
- `model_defaults.spec_heavy`: `opus-4.7:1m@max`.
- `model_defaults.dev`: `sonnet-4.6@high` with `allow_silent_upgrade=false`.
- `model_defaults.review`: `codex-5.5:high@normal`.
- `model_defaults.subagent_fast`: `haiku-4.5@normal`.
- Keep legacy fields updated from `model_defaults` for older commands and templates.
- Keep `builder` as the execution surface, such as `claude-code`, and keep `implementation_model` as the actual model descriptor.
- Keep `reviewer` separate from both implementation model and `review_model`.

## Subagent auto-routing

Use subagent routing when the user wants implementation work divided into task slices or when `subagent_routing.mode = "auto"` in `.cross-review/config.json`.

Default role map:

- scout -> `mmr-context-scout`, model key `subagent_fast`
- worker -> `mmr-implementation-worker`, model key `dev`
- heavy planner -> `mmr-heavy-planner`, model key `spec_heavy`
- review checker -> `mmr-review-checker`, model key `review` only for Claude-compatible review models, otherwise `dev`

Selection rules:

- Choose `scout` for read-only codebase discovery, dependency mapping, and compact context summaries.
- Choose `heavy planner` for cross-cutting design, migrations, security-sensitive changes, unclear requirements, and large Spec Kit decomposition.
- Choose `worker` for scoped code and test edits after the task slice is clear.
- Choose `review checker` for local read-only preflight after an implementation slice or before exporting a review package.

When Claude Code can pass a per-invocation model to the Agent tool, use the resolved model for that subagent call. Otherwise, prefer project-local `.claude/agents/mmr-*.md` overrides generated by `/multi-model-review:cross-review init --subagents auto`; project agents override plugin agents.

Do not write Codex, Gemini, or unknown provider IDs into Claude Code subagent frontmatter unless the local Claude Code installation explicitly supports that custom model. Preserve those models in handoff metadata and use the existing CLI/package path.

## Work-assist orchestration (omo · UltraWork · lazycodex · Superpowers)

`multi-model-review` runs better when heavy, multi-step work is planned and distributed instead of carried by one linear pass. When these tools are installed and callable, use them as the **preferred orchestration and methodology layer**. When they are not, fall back to this plugin's own `mmr-*` subagents (see "Subagent auto-routing") or a plain sequential pass, and record the fallback reason in the completion report. They are work-assist tools, **not** token-savers: only RTK and Headroom report measured savings, and the footer lists these four as usage only.

Roles:

- **Superpowers** (methodology framework): `brainstorming` -> `writing-plans` -> `executing-plans`, plus `test-driven-development`, `systematic-debugging`, `subagent-driven-development`, and `verification-before-completion`. Use it to structure a flow before acting — tighten an ambiguous spec brief, plan implementation before edits, and drive remediation with verification instead of guesswork.
- **UltraWork** (`/ulw <task>`, `ultrawork-sanguo`): a dispatcher that auto-routes independent task slices to specialized agent tiers. Use it to split work that has clear, independent slices.
- **omo** (oh-my-openagent): a multi-agent execution harness with model routing and LSP/AST code intelligence. Use it to run research, implementation, and verification as parallel subagents over the codebase.
- **lazycodex** (omo's Codex edition): LSP/AST code exploration and project memory. Prefer it when the builder or reviewer surface is Codex.

Where to apply:

- **Spec handoff**: use Superpowers `brainstorming`/`writing-plans` to tighten the brief and task slices, and `/ulw` or omo to research existing `spec.md`/`plan.md`/`tasks.md` and project rules. Do not run the spec author model itself.
- **Export (review package)**: use omo or lazycodex for fast code exploration while reducing the package; an `mmr-review-checker` preflight can run under omo or `/ulw`.
- **Ingest (apply review)**: use Superpowers `systematic-debugging` and `subagent-driven-development` to plan accepted fixes, `/ulw` to batch independent findings, and omo to implement-and-verify. Keep every user confirmation gate intact.
- **cross-review (config/status)**: usually no orchestration; just record availability.

Rules:

- Probe availability first (for example `/ulw` or `ultrawork` help, the omo and lazycodex CLIs, the Superpowers skills). If a tool is missing or unusable, fall back without stopping and note "not used / fallback reason".
- Do not pass secrets, credentials, raw `.env` content, or sensitive paths into any of these tools' prompts. Send masked summaries and verification commands only.
- These tools do not reduce tokens. Their large outputs still pass through RTK and Headroom, and the footer reports them as usage only.
- Keep top-level control in the main conversation or slash command: these orchestrators and the `mmr-*` subagents should not spawn deeper orchestration loops.

## Headroom-aware context compression

Use Headroom as an optional context-compression layer for large review inputs when the current host exposes Headroom MCP tools. This is additive to the compact-first package design; it does not replace the diff manifest, focused excerpts, or reviewer context-sufficiency checks.

Headroom concepts to preserve in this skill:

- Content routing: choose different compression behavior for JSON/tool output, logs, diffs, code, search results, and prose.
- Structure preservation: keep file paths, line numbers, function and class signatures, public API names, errors, IDs, hashes, and acceptance criteria visible.
- CCR retrieval: compressed blocks may include a retrieval hash so the reviewer can request the original through `headroom_retrieve` when the same local Headroom store is available.
- Observability: `headroom_stats` can report savings and retrieval counts when available.
- Local-first behavior: do not send raw package material to a hosted compressor and do not start long-running Headroom proxy or wrapper processes from the slash command.

Recommended config:

```json
{
  "context_compression": {
    "provider": "headroom",
    "mode": "auto",
    "min_tokens": 2000,
    "use_mcp": true,
    "require_retrieval_access": false
  }
}
```

Modes:

- `auto`: use Headroom MCP compression when it is callable; otherwise fall back to manual summaries.
- `off`: do not use Headroom for package generation.
- `required`: abort package generation if Headroom MCP compression is not callable.

Headroom is most useful for oversized raw blocks left after manual reduction: verbose `git diff`, logs, JSON tool output, search results, long rules files, large source excerpts, and RAG-like context dumps. Do not use Headroom as a reason to omit review-critical evidence from the manifest or focused excerpts. If the reviewer cannot access the same local Headroom store, treat CCR hashes as optional hints and keep the package independently reviewable.

## Externalized review state (Harness-1 pattern)

The `.cross-review/` directory is a state-externalizing harness in the sense of
[pat-jj/harness-1](https://github.com/pat-jj/harness-1) ("Harness-1:
Reinforcement Learning for Search Agents with State-Externalizing Harnesses",
arXiv:2606.02373): recoverable state — candidate context, curated evidence,
evidence links, verification records, and budget-aware context — lives in
files, while each model keeps only the semantic decisions (what to inspect,
what to curate into the package, which claims to verify, and when the evidence
is sufficient). Apply it concretely:

- **Curated evidence over raw dumps**: the review package is the curated
  evidence set. Judge it by whether the evidence a reviewer needs made it in
  (recall of review-critical context), not by raw size.
- **Evidence links**: every finding should carry `location` plus an `evidence`
  pointer to material reachable inside the package (diff excerpt, manifest
  entry, or spec/plan/tasks/rules line). Ungroundable claims get lower
  confidence, not invented support.
- **Verification records**: during ingest, write each finding's outcome
  (`accepted`, `rejected`, `applied`, `skipped`, plus the verifying command or
  a one-line reason) to `verification_records` in the package `metadata.json`
  so a later session can recover the review state from files alone.
- **Budget-aware context**: compact-first reduction plus optional Headroom
  compression keep the package inside the reviewer's context budget;
  recoverable originals stay in local files and the CCR store, never only in
  conversation history.
- **Sufficiency as an explicit decision**: `Context sufficiency` is the
  "is the evidence sufficient" gate. Reviewers escalate to
  `needs-full-package` instead of guessing; builders rerun with `--full` or
  `--paths` instead of re-reviewing blindly.

Keep secrets out of state files. This layer is workflow design, not a
compression layer: the completion token report still measures RTK and Headroom
only.

## Completion token report (RTK + Headroom)

Every `multi-model-review` flow ends by telling the user how many tokens the two compression layers actually saved. RTK reduces the shell output gathered while building a handoff (`git diff`, `git log`, grep, rules). Headroom compresses the large residual blocks that survive manual reduction. Report the two layers **separately** so the user can see what each one contributed; this is the same measured-reporting contract the companion `token-saver` skill uses.

Measure, never estimate:

- **RTK**: read `rtk gain` (or `rtk gain --format json`). RTK counts as `used=yes` only when shell output actually went through `rtk ...` calls or an RTK hook. If only the built-in Read/Grep/Glob tools ran, or RTK is not installed, report `used=no` and name the next command that would apply, such as `rtk git diff` or `rtk grep`.
- **Headroom**: sum the per-block `original_tokens` and `compressed_tokens` already recorded in this run's `metadata.json` `compressed_blocks`, and/or read session totals from `headroom_stats`. If Headroom was `off`, unavailable, or compressed nothing, report `used=no`.
- If a source has no real statistics, write `used=no` (or `n/a`) with a one-line reason. Do not fabricate a savings number, and do not blend the two layers into a single figure except as an explicit combined total when both are measured.
- Keep secrets, sensitive paths, and raw evidence out of the report.

The work-assist tools (omo, UltraWork, lazycodex, Superpowers) and subagent routing (scout, worker, heavy-planner, review-checker) are orchestration, not a compression layer, so report them as usage only, never as a token-savings number.

Append this block at the very end of the final response for each completed flow (spec handoff written, review package exported, or review report ingested):

```text
**Token, Headroom & RTK**
- **RTK**: used=<yes|no> — saved ≈ <N> tok (<P>%) · via `rtk gain`
- **Headroom**: used=<yes|no> — saved ≈ <N> tok (<P>%) · via `headroom_stats` / package `compressed_blocks`
- **Combined saved**: ≈ <RTK+Headroom> tok   (only when both layers are measured)
- **Work-assist** (orchestration, usage only): ulw=<used|n/a>, omo=<used|n/a>, lazycodex=<used|n/a>, superpowers=<used|n/a>
- **Subagent routing** (usage only): scout=<used|n/a>, worker=<used|n/a>, heavy-planner=<used|n/a>, review-checker=<used|n/a>
```

When neither layer engaged — for example a small config-only `cross-review status` call — still print the block with `used=no` and the next command that would apply, so the report stays consistent across every flow.

## Spec handoff flow

Use `/multi-model-review:spec-handoff` before implementation when the user wants another model to produce or refine Spec Kit artifacts.

When the work-assist orchestration layer is available, drive this flow through it (Superpowers to shape the brief, `/ulw` or omo to research artifacts); otherwise fall back to the `mmr-*` subagents or a sequential pass.

### 1. Resolve model routing

- Read `model_defaults`, `spec_author_model`, `spec_author_options`, optional `spec_author_profile`, `implementation_model`, and `implementation_options` from `.cross-review/config.json`.
- If the user passes `--spec-model <model[:axis]@axis>`, use it for this handoff only.
- If the user passes `--heavy`, use `model_defaults.spec_heavy`.
- If the user passes `--plan`, keep using `/multi-model-review:spec-handoff` but emphasize `plan.md` and `tasks.md` output for implementation handoff planning.
- If both `--spec-model` and `--heavy` are present, the explicit `--spec-model` wins and you should say so.
- If `model_defaults` is missing, infer it from legacy fields:
  - `codex-5.5` or `gpt-5.5` -> `{ "intelligence": "very-high", "reasoning": "xhigh", "speed": "normal" }`
  - `opus-4.7`, `claude-opus-4.7`, or `claude-opus-4.7-1m` -> `{ "context": "1M", "workload": "max" }`
- If only legacy `spec_author_profile` is present, preserve it and add a package note that options were inferred from the display profile when possible.
- If `implementation_options` is missing for a Claude implementation model, infer `{ "workload": "high", "allow_silent_upgrade": false }`.
- If the user asks for Opus heavy mode, use `opus-4.7:1m@max` unless the user gives different model options.
- Derive `spec_author_profile` from `spec_author_options` for human-readable output, for example `intelligence=very-high, reasoning=xhigh, speed=normal`.
- Resolve `subagent_routing` from config. If absent, treat it as `off` unless the user explicitly asks for subagent splitting.
- If subagent routing is enabled, include the role map and selection rules in `SUBAGENT_ROUTING`; do not invoke subagents while creating the spec handoff.

### 2. Gather spec inputs

- User feature brief from the command arguments or conversation
- Existing `spec.md`, `plan.md`, and `tasks.md` when refining an existing Spec Kit feature
- Relevant project rules, README, and touched files when they affect constraints
- Any explicit non-goals, acceptance criteria, deadlines, or integration boundaries

### 3. Render the spec-authoring template

Use `templates/spec-authoring-prompt.md`.

The template must tell the spec author model to:

- produce or update `spec.md`, `plan.md`, and `tasks.md`
- optimize the artifacts for implementation by `implementation_model`
- create token-conscious task slices suitable for Claude Sonnet 4.6 when that is the implementation model
- add route hints to `tasks.md` such as `[route:scout]`, `[route:heavy-planner]`, `[route:worker]`, or `[route:review-checker]` when subagent routing is enabled
- avoid implementing code
- output file blocks with target paths

### 4. Write outputs

Write:

- `spec-authoring-prompt.md`
- `metadata.json` with the selected detailed spec model, spec author model/options/profile, selected detailed implementation model, implementation options, resolved subagent routing, timestamp, slug, original arguments, and source notes

Print the selected spec author model, implementation model, subagent routing mode, prompt path, and expected `spec-output.md` path.

Finish the response with the **Completion token report** described above.

## Export flow

Build a self-contained markdown package. Default to **compact-first** packaging with optional Headroom-aware compression for large residual context.

When the work-assist orchestration layer is available, use omo or lazycodex for fast code exploration while reducing the package; otherwise proceed natively.

### 1. Resolve the feature

- If the user gave a slug, use `specs/<slug>/`
- Else pick the most recently modified `specs/*/`
- If no Spec Kit directory exists, fall back to ad-hoc diff-only mode

### 2. Resolve scope

- Base ref: config value or `main`
- Optional path filter from `--paths`
- Optional `--review-model <model[:axis]@axis>` override for a single review package
- Optional `--headroom auto|off|required` override for a single review package
- Reviewer mode:
  - `codex-mcp` implies `micro`
  - `codex-auto`, `codex-cli`, `claude-code`, `gemini-cli`, `hermes` should default to `compact`
  - `--full` overrides to `full`

### 3. Resolve context compression

- Read `context_compression` from config; default to Headroom `auto`.
- If the user passes `--headroom`, it overrides config for this package only.
- Prefer Headroom MCP tools when `headroom_compress`, `headroom_retrieve`, and `headroom_stats` are callable.
- Treat the Headroom CLI as an install/status surface unless the current host exposes callable MCP tools; do not assume a direct CLI compression command.
- If mode is `required` and Headroom MCP compression is unavailable, abort with a short reason.
- If mode is `auto` and Headroom is unavailable, continue with manual RTK-style reduction.

### 4. Gather raw inputs

- `spec.md`
- `plan.md`
- `tasks.md`
- `git diff <base>...HEAD`
- `git log <base>...HEAD --oneline`
- relevant `CLAUDE.md` files

### 5. Reduce the package before rendering

Use RTK-style reduction first:

- **Filtering**: drop generated noise, lockfile churn, snapshots, vendored paths unless the feature depends on them
- **Grouping**: create a grouped diff manifest before raw hunks
- **Truncation**: cap logs, long task lists, repeated rule text, and low-signal hunks
- **Deduplication**: collapse repeated context

Then, when Headroom is enabled and callable, compress large remaining blocks using `headroom_compress`. Keep the compressed content in the package, not the raw block, and record:

- source label
- original and compressed token estimates when available
- savings percentage when available
- transforms when available
- CCR retrieval hash when available
- query hints that help retrieve a smaller slice, such as file path, symbol, task ID, or error text

If a Headroom-compressed block hides context that is necessary to verify a finding, the reviewer should either use `headroom_retrieve` when available or set `Context sufficiency: needs-full-package`.

Create these placeholders:

- `CONTEXT_COMPRESSION`
- `SPEC_AUTHOR_MODEL`
- `SPEC_AUTHOR_OPTIONS`
- `SPEC_AUTHOR_PROFILE`
- `IMPLEMENTATION_MODEL`
- `IMPLEMENTATION_OPTIONS`
- `SUBAGENT_ROUTING`
- `REVIEW_MODEL`
- `REVIEW_OPTIONS`
- `REVIEW_PROFILE`
- `SPEC_BRIEF`
- `PLAN_BRIEF`
- `TASKS_BRIEF`
- `RULES_BRIEF`
- `LOG_BRIEF`
- `DIFF_MANIFEST`
- `DIFF_EXCERPTS`
- `PACKAGE_NOTES`

Appendix placeholders:

- `SPEC_APPENDIX`
- `PLAN_APPENDIX`
- `TASKS_APPENDIX`
- `RULES_APPENDIX`
- `RAW_DIFF_APPENDIX`

Rules:

- In `compact` and `micro`, appendices should be short omission notes, not raw dumps.
- In `full`, appendices may include the raw source material.
- If RTK is installed, prefer RTK-assisted summaries when gathering shell output. If it is not installed, create equivalent summaries manually.
- If Headroom is available, use it only after manual reduction and never as the sole holder of review-critical evidence for reviewers that cannot access the same local CCR store.

### 6. Render the reviewer template

Use `templates/<reviewer>-review-prompt.md`.

The template must tell the reviewer to:

- validate spec-to-code alignment
- find real correctness, security, and rule issues
- score confidence 0-100
- report `Context sufficiency`
- write `review-report.md` using `templates/review-report.md`

### 7. Write outputs

Write:

- `review-package.md`
- `metadata.json` with base/head, roles, selected detailed review model, spec author model, spec author options, implementation model, implementation options, subagent routing, context compression mode, Headroom availability, compressed block metadata, profile, and truncation notes

### 8. Print the next command

Print the exact reviewer command and the output path for `review-report.md`. For Codex review models, include the selected model ID in the command hint and preserve `speed=priority` in metadata even if the local CLI requires a separate priority mechanism.

If the report later says `Context sufficiency: needs-full-package`, tell the user to rerun:

- `/multi-model-review:review-package --full`
- or `/multi-model-review:review-package --paths <subset>`

Finish the response with the **Completion token report**. This is the flow with the most concrete numbers: sum the `compressed_blocks` savings recorded in `metadata.json` for the Headroom line, and read `rtk gain` for the RTK line.

## Ingest flow

Read the latest `review-report.md` unless the user points to a specific package directory.

When the work-assist orchestration layer is available, use Superpowers and `/ulw` or omo to plan and apply accepted fixes; otherwise fall back to the `mmr-*` subagents. Keep every user confirmation gate intact.

Parse:

- `Context sufficiency`
- `Verdict`
- `Summary`
- `Findings`

Default handling:

- drop findings with confidence below 70
- group remaining findings by severity and file
- do not edit until the user accepts a finding

Special handling:

- if `Context sufficiency = needs-full-package`, stop before making broad edits and recommend a `--full` or `--paths` rerun
- never auto-apply a `critical` finding without explicit approval

After summarizing applied versus skipped findings, write each finding's outcome (`accepted`, `rejected`, `applied`, or `skipped`, plus the verifying command or a one-line reason) to `verification_records` in the package `metadata.json` (Harness-1 verification records), then finish the response with the **Completion token report**.

## Important constraints

- Do not run the reviewer model for the user.
- Do not package secrets.
- Do not dump raw artifacts in compact mode just because they are available.
- Preserve the reviewer's language when quoting findings back to the user.
- Do not silently upgrade the configured implementation model. If `model_defaults.dev` says `sonnet-4.6@high`, keep that routing unless the user explicitly overrides it.
- Automatic subagent routing may choose a different configured role model for a different kind of task, but record the selected role, agent, model key, and reason in metadata or the task plan.
- Subagents should not spawn subagents. Keep orchestration in the main conversation or slash command.
- End every completed flow with the Completion token report. Report RTK and Headroom savings separately, from `rtk gain` and `headroom_stats`/`compressed_blocks` measurements only; never estimate a number, and mark `used=no` when a layer did not engage.
- Treat the work-assist orchestration layer (omo, UltraWork, lazycodex, Superpowers) as optional. Use it when available, fall back to the `mmr-*` subagents or sequential execution otherwise, report it as usage only (never a savings number), and never pass secrets into its prompts.
- Keep review state externalized and recoverable (Harness-1): the package directory plus `metadata.json` (including `verification_records`) is the durable record, and a new session resumes from those files, not from conversation memory.

## Related files

- `commands/cross-review.md`
- `commands/spec-handoff.md`
- `commands/review-package.md`
- `commands/apply-review.md`
- `templates/spec-authoring-prompt.md`
- `templates/review-report.md`
- `templates/hermes-review-prompt.md`
- `agents/mmr-context-scout.md`
- `agents/mmr-implementation-worker.md`
- `agents/mmr-heavy-planner.md`
- `agents/mmr-review-checker.md`
