# Token Efficiency

`multi-model-review` sits on top of Spec Kit, but cross-model review has a different bottleneck than implementation: the reviewer pays for every token in the handoff package. The workflow also separates high-reasoning spec authoring from token-heavy implementation so each stage can use the most appropriate model.

This document summarizes the token-efficiency changes added here and how they map to ideas from [rtk](https://github.com/rtk-ai/rtk) and [Headroom](https://github.com/chopratejas/headroom).

## Problem

The original review package model was simple:

- include `spec.md`
- include `plan.md`
- include `tasks.md`
- include relevant `CLAUDE.md`
- include `git log`
- include the full `git diff`

That works for small reviews, but cost grows quickly with:

- repeated boilerplate
- generated files
- lockfile churn
- repeated rule text
- low-signal diff hunks

## RTK ideas applied here

RTK reduces shell-output tokens with four main moves:

- filtering
- grouping
- truncation
- deduplication

`multi-model-review` now applies the same pattern to review packages.

## Headroom ideas applied here

Headroom compresses tool outputs, logs, files, RAG chunks, and conversation context before they reach an LLM. Its useful ideas for `multi-model-review` are:

- content-aware routing for JSON/tool output, logs, diffs, code, search results, and prose
- structure preservation for identifiers, file paths, line numbers, signatures, errors, and acceptance criteria
- CCR-style retrieval markers so compressed content can be expanded later when the same local Headroom store is available
- compression stats so the package can report token savings and retrieval hints

`multi-model-review` treats Headroom as optional. The default review package is still compact-first and self-contained enough for a reviewer to decide whether context is sufficient. When Headroom MCP tools are available, the package generator may compress large raw blocks that remain after manual filtering, grouping, truncation, and deduplication.

Headroom modes:

| Mode | Behavior |
|------|----------|
| `auto` | Use Headroom MCP compression when callable; otherwise continue with manual summaries |
| `off` | Do not use Headroom |
| `required` | Abort package generation if Headroom MCP compression is unavailable |

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

## Package profiles

Three profiles are expected:

| Profile | Use case | Default content |
|---------|----------|-----------------|
| `micro` | MCP-size review | briefs + tiny diff excerpts |
| `compact` | normal default | briefs + manifest + focused excerpts |
| `full` | escalation pass | raw appendices included |

Recommended defaults:

- development spec author -> `codex-5.5:xhigh@normal`
- heavy spec author -> `opus-4.7:1m@max`
- implementation -> `sonnet-4.6@high` with silent upgrade disabled
- review -> `codex-5.5:high@normal`
- fast subagent scout -> `haiku-4.5@normal`
- subagent routing -> `auto` with `balanced` policy
- `codex-mcp` -> `micro`
- `codex-auto` -> `compact`
- `codex-cli` -> `compact`
- `claude-code` reviewer -> `compact`
- `gemini-cli` reviewer -> `compact`
- `hermes` reviewer -> `compact`
- Headroom context compression -> `auto`

## Compact-first workflow

1. export a compact package
2. ask the reviewer for `Context sufficiency`
3. rerun with `--full` only if needed

This is the review-package equivalent of RTK's "compact first, raw when necessary" philosophy.

## Spec and implementation routing

For development itself, use a separate spec-authoring handoff before implementation:

1. create `spec-authoring-prompt.md` with `/multi-model-review:spec-handoff`
2. run `codex-5.5:xhigh@normal`, or use `--heavy` for `opus-4.7:1m@max`
3. apply the returned `spec.md`, `plan.md`, and `tasks.md`
4. use route hints such as `[route:scout]`, `[route:heavy-planner]`, `[route:worker]`, and `[route:review-checker]` when task slices should be delegated
5. implement with `sonnet-4.6@high` unless the user explicitly overrides the dev model

This keeps the expensive reasoning pass focused on durable artifacts and keeps the implementation pass bounded by small, explicit tasks.

Subagent routing is also a token-efficiency tool:

- `mmr-context-scout` keeps broad exploration out of the main context
- `mmr-heavy-planner` spends the high-reasoning budget only on ambiguous or cross-cutting design
- `mmr-implementation-worker` receives scoped tasks instead of the whole project history
- `mmr-review-checker` does local preflight before a compact external review package is exported

## What stays in compact mode

Keep:

- objective and acceptance criteria
- architecture and risk summary
- tasks tied to touched files
- only the rules relevant to touched paths
- grouped file manifest
- focused diff excerpts
- explicit omission notes
- Headroom compressed-block notes with retrieval hashes and query hints, when available

Usually omit:

- raw lockfile churn
- generated bundles
- vendored code
- snapshots
- repeated boilerplate
- appendices that do not change the review decision

## Headroom package notes

When Headroom is applied, `PACKAGE_NOTES` should list each compressed block:

- source label, such as `git-diff`, `rules`, `tasks`, `log`, or `source-excerpt`
- original and compressed token estimates when available
- savings percentage and transforms when available
- CCR hash when available
- retrieval query hints, such as file path, symbol, task ID, or error text

Do not rely on a transient CCR hash as the only copy of evidence that an external reviewer needs. If the reviewer cannot access the same local Headroom MCP/proxy store, the package should still contain enough manifest and focused excerpts for at least a `limited-but-actionable` review. Otherwise the reviewer should return `Context sufficiency: needs-full-package`.

## Why `Context sufficiency` matters

Compact packages only work if the reviewer can say:

- `sufficient`
- `limited-but-actionable`
- `needs-full-package`

That field prevents the reviewer from inventing confidence when the compact package hides too much.

## Optional RTK usage

The plugin does not require RTK, but RTK can still help when available:

- use RTK-assisted summaries when exploring large diffs
- keep the same reduction mindset for shell output around the review loop
- prefer compact-first inspection before falling back to raw output

## Optional Headroom usage

The plugin does not require Headroom. Use it only when the current host exposes Headroom MCP tools such as `headroom_compress`, `headroom_retrieve`, and `headroom_stats`.

- Use `--headroom auto` for normal package generation.
- Use `--headroom off` when the reviewer will not share the local retrieval store and the package must avoid retrieval markers.
- Use `--headroom required` only when the workflow depends on Headroom MCP retrieval and should fail fast if it is unavailable.

The Headroom CLI is useful for install and status checks such as `headroom --version` and `headroom mcp status`, but do not assume a direct CLI compression subcommand exists.

## Completion token report

Token efficiency is only visible if the workflow reports it. Every `multi-model-review` flow therefore ends with a measured **Token, Headroom & RTK** footer that states what each compression layer saved, separately:

- **RTK** covers the shell output gathered while building a handoff (`git diff`, `git log`, grep, rules). Numbers come from `rtk gain`.
- **Headroom** covers the large residual blocks compressed during packaging. Numbers come from the package `metadata.json` `compressed_blocks` (`original_tokens` vs `compressed_tokens`) and/or `headroom_stats`.

The two layers are always reported separately and from real statistics only. If a layer did not engage — RTK not installed, Headroom `off` or unavailable, or a config-only call that compressed nothing — the report says `used=no` with the next command that would apply, instead of estimating a number. The work-assist orchestration tools (omo, UltraWork, lazycodex, Superpowers) and subagent routing are listed as usage only, because they are orchestration layers rather than compression layers.

Footer shape:

```text
**Token, Headroom & RTK**
- **RTK**: used=<yes|no> — saved ≈ <N> tok (<P>%) · via `rtk gain`
- **Headroom**: used=<yes|no> — saved ≈ <N> tok (<P>%) · via `headroom_stats` / package `compressed_blocks`
- **Combined saved**: ≈ <RTK+Headroom> tok   (only when both layers are measured)
- **Work-assist** (orchestration, usage only): ulw=<used|n/a>, omo=<used|n/a>, lazycodex=<used|n/a>, superpowers=<used|n/a>
- **Subagent routing** (usage only): scout=<used|n/a>, worker=<used|n/a>, heavy-planner=<used|n/a>, review-checker=<used|n/a>
```

This mirrors the measured-reporting contract of the companion `token-saver` skill: only RTK and Headroom report token savings, and they are never blended or estimated. The work-assist orchestration layer (omo, UltraWork, lazycodex, Superpowers) helps run the work but never reports a savings number.

## References

- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [rtk](https://github.com/rtk-ai/rtk)
- [Headroom](https://github.com/chopratejas/headroom)
