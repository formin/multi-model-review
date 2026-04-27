# multi-model-review

Multi-model code review and model-routed Spec-Driven Development.

Write specs with one model, implement with another, and review with a different one. This plugin turns [GitHub Spec Kit](https://github.com/github/spec-kit) artifacts (`spec.md`, `plan.md`, `tasks.md`) plus your git diff into portable handoffs that any CLI-capable model can consume.

The spec author or reviewer does not need to share your current session. Each handoff lives on disk as markdown, and you run the other model yourself.

## Why multi-model?

A model reviewing its own output often rationalizes. A different model brings:

- different priors
- different bug sensitivity
- different style and rule interpretation

Spec Kit is excellent at the single-agent flow:

- `/speckit.specify`
- `/speckit.plan`
- `/speckit.tasks`
- `/speckit.implement`

`multi-model-review` is the cross-model loop around that flow:

- a spec-authoring model creates or refines `spec.md`, `plan.md`, and `tasks.md`
- the implementation model builds the feature
- the builder creates a review package
- the reviewer reads that package and writes `review-report.md`
- the builder ingests the report and applies fixes

## Recommended model routing

| Stage | Default | Notes |
|-------|---------|-------|
| Spec authoring | `codex-5.5` | Use `intelligence=very-high, speed=normal` |
| Spec authoring alternative | `opus-4.7` | Use `context=1M, workload=high` |
| Implementation | `claude-sonnet-4.6` | Use `workload=high` for token-heavy development work |
| Review | `codex-auto` | Compact package first, CLI fallback for larger reviews |

The routing is stored in `.cross-review/config.json`, so the skill can keep the spec-writing model, model-specific options, and the implementation model distinct throughout packaging and review.

Example spec author options:

```json
{
  "spec_author_model": "codex-5.5",
  "spec_author_options": {
    "intelligence": "very-high",
    "speed": "normal"
  }
}
```

```json
{
  "spec_author_model": "opus-4.7",
  "spec_author_options": {
    "context": "1M",
    "workload": "high"
  }
}
```

UI option mapping:

| UI | Config key | Values |
|----|------------|--------|
| Codex model | `spec_author_model` | `codex-5.5` or `gpt-5.5` for the GPT-5.5 UI entry |
| Codex intelligence | `intelligence` | `low`, `medium`, `high`, `very-high` |
| Codex speed | `speed` | `normal`, `fast` |
| Claude model | `spec_author_model` or `implementation_model` | `claude-opus-4.7`, `claude-opus-4.7-1m`, `claude-sonnet-4.6`, `claude-haiku-4.5` |
| Claude context | `context` | `standard`, `1M` |
| Claude workload | `workload` | `low`, `normal`, `high` |

## Compact-first packaging

The package format is now compact-first, inspired by [rtk](https://github.com/rtk-ai/rtk).

Instead of dumping every artifact in full, the default package starts with:

- `SPEC_BRIEF`
- `PLAN_BRIEF`
- `TASKS_BRIEF`
- `RULES_BRIEF`
- `DIFF_MANIFEST`
- `DIFF_EXCERPTS`

This keeps the default handoff much smaller. If the reviewer needs more context, rerun with `--full` or narrow the scope with `--paths`.

Package profiles:

| Profile | Use case |
|---------|----------|
| `micro` | sub-60 second MCP reviews |
| `compact` | default |
| `full` | escalation pass when compact is insufficient |

Details live in [docs/TOKEN_EFFICIENCY.md](docs/TOKEN_EFFICIENCY.md).

## Supported reviewers

| Reviewer | Template | Example command |
|----------|----------|-----------------|
| Claude | `templates/claude-review-prompt.md` | `claude -p "$(cat <pkg>/review-package.md)" > <pkg>/review-report.md` |
| Codex (auto) | `templates/codex-auto-review-prompt.md` | Try MCP first, fall back to CLI on timeout |
| Codex (CLI) | `templates/codex-cli-review-prompt.md` | `codex exec --file <pkg>/review-package.md > <pkg>/review-report.md` |
| Codex (MCP) | `templates/codex-mcp-review-prompt.md` | Inline MCP call for tiny packages |
| Gemini | `templates/gemini-review-prompt.md` | `gemini --file <pkg>/review-package.md > <pkg>/review-report.md` |

Adding a reviewer is still a file-only change: add `templates/<reviewer>-review-prompt.md`.

## Codex modes

The `mcp__codex__codex` path is hard-limited to about 60 seconds, so non-trivial reviews should use the CLI path.

| Value | Execution method | Best for |
|-------|------------------|----------|
| `codex-mcp` | inline MCP | tiny validations |
| `codex-cli` | `codex exec --file ...` | longer reviews |
| `codex-auto` | MCP first, CLI fallback | default |

## Requirements

- [Claude Code](https://claude.com/claude-code) 1.x
- `git` on `PATH`
- a reviewer model installed locally:
  - [Codex CLI](https://github.com/openai/codex)
  - [Gemini CLI](https://github.com/google-gemini/gemini-cli)
  - another Claude session
  - any other CLI-capable LLM with a matching template
- a Spec Kit project is recommended, but not required

## Install

Inside a Claude Code session:

```text
/plugin install multi-model-review@multi-model-review
```

Or at the OS shell:

```bash
claude plugin install multi-model-review@multi-model-review
claude plugin list
```

Then run `/reload-plugins` or restart Claude Code.

## Quick start

Inside your project:

```text
/multi-model-review:cross-review init
```

Typical answers:

- `builder = claude-code`
- `spec_author_model = codex-5.5`
- `spec_author_options = {"intelligence":"very-high","speed":"normal"}`
- `spec_author_profile = intelligence=very-high, speed=normal`
- `implementation_model = claude-sonnet-4.6`
- `implementation_options = {"workload":"high"}`
- `reviewer = codex-auto`
- `base ref = main`
- `package profile = compact`

Create a spec-authoring handoff when you want the configured spec model to write or refine the development artifacts:

```text
/multi-model-review:spec-handoff 001-auth-rework
```

Then implement your feature as usual and package it:

```text
/multi-model-review:review-package
```

The plugin writes:

```text
.cross-review/packages/<timestamp>-<slug>/
  review-package.md
  metadata.json
```

Run the reviewer in a separate terminal. Example:

```bash
PKG=.cross-review/packages/20260421-1400-auth-rework
codex exec --file $PKG/review-package.md > $PKG/review-report.md
```

Then ingest the report:

```text
/multi-model-review:apply-review
```

If the reviewer reports `Context sufficiency: needs-full-package`, rerun:

```text
/multi-model-review:review-package --full
```

Or scope it tighter:

```text
/multi-model-review:review-package --paths src/auth,src/api
```

## Basic usage examples

### Codex 5.5 writes specs, Claude Sonnet 4.6 implements

Use this when you want Codex to spend the deep reasoning budget on durable Spec Kit artifacts, then keep implementation on Sonnet 4.6.

```json
{
  "builder": "claude-code",
  "spec_author_model": "codex-5.5",
  "spec_author_options": {
    "intelligence": "very-high",
    "speed": "normal"
  },
  "implementation_model": "claude-sonnet-4.6",
  "implementation_options": {
    "workload": "high"
  },
  "reviewer": "codex-auto",
  "base_ref": "main",
  "package_profile": "compact"
}
```

Then run:

```text
/multi-model-review:spec-handoff 001-auth-rework
/multi-model-review:review-package 001-auth-rework
/multi-model-review:apply-review
```

### Opus 4.7 1M writes specs, Claude Sonnet 4.6 implements

Use this when the spec pass needs the Claude 1M context option and high workload setting.

```text
/multi-model-review:spec-handoff 001-auth-rework --model claude-opus-4.7-1m --model-option context=1M --model-option workload=high --implementation-model claude-sonnet-4.6 --implementation-option workload=high
```

Equivalent config:

```json
{
  "builder": "claude-code",
  "spec_author_model": "claude-opus-4.7-1m",
  "spec_author_options": {
    "context": "1M",
    "workload": "high"
  },
  "implementation_model": "claude-sonnet-4.6",
  "implementation_options": {
    "workload": "high"
  },
  "reviewer": "codex-auto",
  "base_ref": "main",
  "package_profile": "compact"
}
```

## Commands

| Command | Purpose |
|---------|---------|
| `/multi-model-review:cross-review [init|status]` | configure model routing, roles, and defaults |
| `/multi-model-review:spec-handoff [slug|brief] [--model <id>] [--model-option <key=value>] [--implementation-model <id>] [--implementation-option <key=value>]` | export the development spec authoring handoff |
| `/multi-model-review:review-package [slug] [--base <ref>] [--full] [--paths <glob,...>]` | export the reviewer handoff |
| `/multi-model-review:apply-review [path] [--min-confidence N]` | ingest the report and apply fixes |

## Review report contract

All reviewer templates point at the same schema in [templates/review-report.md](templates/review-report.md).

The report now includes:

- `Context sufficiency`
- `Verdict`
- `Summary`
- `Findings`

That extra `Context sufficiency` field is what makes compact-first safe: the reviewer can explicitly ask for a fuller package instead of guessing.

## Design choices

- **No side calls.** You run the reviewer yourself.
- **Markdown interchange only.** Packages and reports are plain files.
- **Compact-first.** Small default package, full package on demand.
- **Confidence-gated.** Findings below 70 are hidden by default.
- **Template-driven.** New reviewers require template files, not code.

## Docs

- [docs/USAGE.md](docs/USAGE.md)
- [docs/EXAMPLE.md](docs/EXAMPLE.md)
- [docs/TOKEN_EFFICIENCY.md](docs/TOKEN_EFFICIENCY.md)

## Relationship to Spec Kit

This project is complementary to Spec Kit, not a fork of it. Spec Kit remains the lifecycle owner for specification, planning, tasks, and implementation. `multi-model-review` adds the cross-model review loop on top.

## License

MIT. See [LICENSE](LICENSE).
