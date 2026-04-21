# Token Efficiency

`multi-model-review` sits on top of Spec Kit, but cross-model review has a different bottleneck than implementation: the reviewer pays for every token in the handoff package.

This document summarizes the token-efficiency changes added here and how they map to ideas from [rtk](https://github.com/rtk-ai/rtk).

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

## Package profiles

Three profiles are expected:

| Profile | Use case | Default content |
|---------|----------|-----------------|
| `micro` | MCP-size review | briefs + tiny diff excerpts |
| `compact` | normal default | briefs + manifest + focused excerpts |
| `full` | escalation pass | raw appendices included |

Recommended defaults:

- `codex-mcp` -> `micro`
- `codex-auto` -> `compact`
- `codex-cli` -> `compact`
- `claude-code` reviewer -> `compact`
- `gemini-cli` reviewer -> `compact`

## Compact-first workflow

1. export a compact package
2. ask the reviewer for `Context sufficiency`
3. rerun with `--full` only if needed

This is the review-package equivalent of RTK's "compact first, raw when necessary" philosophy.

## What stays in compact mode

Keep:

- objective and acceptance criteria
- architecture and risk summary
- tasks tied to touched files
- only the rules relevant to touched paths
- grouped file manifest
- focused diff excerpts
- explicit omission notes

Usually omit:

- raw lockfile churn
- generated bundles
- vendored code
- snapshots
- repeated boilerplate
- appendices that do not change the review decision

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

## References

- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [rtk](https://github.com/rtk-ai/rtk)
