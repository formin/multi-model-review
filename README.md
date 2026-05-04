# multi-model-review

Multi-model code review and model-routed Spec-Driven Development.

Write specs with one model, implement with another, and review with a different one. This plugin turns [GitHub Spec Kit](https://github.com/github/spec-kit) artifacts (`spec.md`, `plan.md`, `tasks.md`) plus your git diff into portable handoffs that any CLI-capable model can consume.

The spec author or reviewer does not need to share your current session. Each handoff lives on disk as markdown, and you run the other model yourself.

## Quick usage

Install as a Spec Kit extension, then initialize model routing in the target repository:

```text
specify extension add --dev /path/to/multi-model-review
/speckit.multi-model-review.cross-review init --spec codex-5.5:xhigh@normal --spec-heavy opus-4.7:1m@max --dev sonnet-4.6@high --review codex-5.5:high@normal --subagents auto
```

Create or refine Spec Kit artifacts, implement the generated tasks, and export a compact review package:

```text
/speckit.multi-model-review.spec-handoff 001-auth-rework --plan
/speckit.implement
/speckit.multi-model-review.review-package 001-auth-rework
```

When subagent routing is enabled, tasks can include route hints so Claude Code chooses the right specialist and model automatically:

```markdown
- [ ] T001 [route:scout] Map the affected files and project rules.
- [ ] T002 [route:heavy-planner] Plan the cross-cutting API and migration changes.
- [ ] T003 [route:worker] Implement the scoped code and tests.
- [ ] T004 [route:review-checker] Run a local read-only preflight before external review.
```

Run the reviewer yourself, save `review-report.md`, then ingest accepted findings:

```text
/speckit.multi-model-review.apply-review
```

Claude Code plugin installs still expose the legacy command names such as `/multi-model-review:cross-review`.

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
| Spec authoring | `codex-5.5:xhigh@normal` | Deep reasoning, normal speed |
| Heavy spec authoring | `opus-4.7:1m@max` | Large audits and 1M-context passes |
| Implementation | `sonnet-4.6@high` | Token-heavy development work, no silent upgrade |
| Review | `codex-5.5:high@normal` | Cost-aware cross-review |
| Fast subagent scout | `haiku-4.5@normal` | Read-only discovery and context summaries |
| Subagent routing | `auto` | Choose scout, planner, worker, or checker by task situation |

The routing is stored in `.cross-review/config.json`, so the skill can keep the spec-writing model, heavy-spec model, implementation model, subagent role models, and review model distinct throughout packaging and review.

Detailed model specs use:

```text
<model>[:<axis-a>][@<axis-b>]
```

Examples:

- `codex-5.5:xhigh@normal`
- `codex-5.5:high@priority`
- `opus-4.7:1m@max`
- `sonnet-4.6@high`

Example spec author options:

```json
{
  "model_defaults": {
    "spec": {
      "raw": "codex-5.5:xhigh@normal",
      "provider": "codex",
      "model": "codex-5.5",
      "reasoning": "xhigh",
      "speed": "normal"
    }
  },
  "spec_author_model": "codex-5.5",
  "spec_author_options": {
    "intelligence": "very-high",
    "reasoning": "xhigh",
    "speed": "normal"
  }
}
```

```json
{
  "model_defaults": {
    "spec_heavy": {
      "raw": "opus-4.7:1m@max",
      "provider": "claude",
      "model": "claude-opus-4.7",
      "context": "1M",
      "workload": "max"
    }
  },
  "spec_heavy_model": "claude-opus-4.7",
  "spec_heavy_options": {
    "context": "1M",
    "workload": "max"
  }
}
```

UI option mapping:

| UI | Config key | Values |
|----|------------|--------|
| Codex model | `spec_author_model` | `codex-5.5` or `gpt-5.5` for the GPT-5.5 UI entry |
| Codex reasoning | `reasoning` | `low`, `medium`, `high`, `xhigh` |
| Codex intelligence | `intelligence` | `low`, `medium`, `high`, `very-high` |
| Codex speed | `speed` | `normal`, `fast`, `priority` |
| Claude model | `spec_author_model` or `implementation_model` | `claude-opus-4.7`, `claude-opus-4.7-1m`, `claude-sonnet-4.6`, `claude-haiku-4.5` |
| Claude context | `context` | `standard`, `1M` |
| Claude workload | `workload` | `low`, `normal`, `high`, `max` |

## Automatic subagent routing

The plugin ships Claude Code subagents in `agents/` and can record a routing policy in `.cross-review/config.json`.

| Subagent | Default model | Use for |
|----------|---------------|---------|
| `mmr-context-scout` | `haiku` | fast read-only discovery, dependency mapping, task context |
| `mmr-heavy-planner` | `opus` | cross-cutting plans, migrations, ambiguous or security-sensitive work |
| `mmr-implementation-worker` | `sonnet` | scoped implementation and tests |
| `mmr-review-checker` | `sonnet` | local read-only preflight before external review |

When `subagent_routing.mode` is `auto`, tasks can be tagged with route hints such as `[route:scout]`, `[route:heavy-planner]`, `[route:worker]`, or `[route:review-checker]`. The main builder chooses the matching subagent and the configured role model. If Claude Code cannot use a configured non-Claude model in subagent frontmatter, the model stays in the markdown handoff or CLI review path instead of being silently substituted.

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
| Codex (CLI) | `templates/codex-cli-review-prompt.md` | `codex exec -m codex-5.5 --file <pkg>/review-package.md > <pkg>/review-report.md` |
| Codex (MCP) | `templates/codex-mcp-review-prompt.md` | Inline MCP call for tiny packages |
| Gemini | `templates/gemini-review-prompt.md` | `gemini --file <pkg>/review-package.md > <pkg>/review-report.md` |

Adding a reviewer is still a file-only change: add `templates/<reviewer>-review-prompt.md`.

## Codex modes

The `mcp__codex__codex` path is hard-limited to about 60 seconds, so non-trivial reviews should use the CLI path.

| Value | Execution method | Best for |
|-------|------------------|----------|
| `codex-mcp` | inline MCP | tiny validations |
| `codex-cli` | `codex exec -m <review_model> --file ...` | longer reviews |
| `codex-auto` | MCP first, CLI fallback | default |

## Requirements

- [Claude Code](https://claude.com/claude-code) 1.x
- `git` on `PATH`
- a reviewer model installed locally:
  - [Codex CLI](https://github.com/openai/codex)
  - [Gemini CLI](https://github.com/google-gemini/gemini-cli)
  - another Claude session
  - any other CLI-capable LLM with a matching template
- a Spec Kit project initialized with `specify init` when installing through `specify extension`

## Install

### Spec Kit Extension

For local development:

```bash
specify extension add --dev /path/to/multi-model-review
specify extension list
```

After a `v0.1.0` release is published, users can install directly from the release archive:

```bash
specify extension add multi-model-review --from https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.0.zip
```

Registered Spec Kit command names:

| Spec Kit command | Legacy Claude plugin command |
|------------------|------------------------------|
| `/speckit.multi-model-review.cross-review` | `/multi-model-review:cross-review` |
| `/speckit.multi-model-review.spec-handoff` | `/multi-model-review:spec-handoff` |
| `/speckit.multi-model-review.review-package` | `/multi-model-review:review-package` |
| `/speckit.multi-model-review.apply-review` | `/multi-model-review:apply-review` |

### Claude Code Plugin

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
/speckit.multi-model-review.cross-review init \
  --spec codex-5.5:xhigh@normal \
  --spec-heavy opus-4.7:1m@max \
  --dev sonnet-4.6@high \
  --review codex-5.5:high@normal \
  --subagents auto
```

This writes `.cross-review/config.json`. The same settings can be changed later with:

```text
/speckit.multi-model-review.cross-review models set --spec codex-5.5:xhigh@normal --spec-heavy opus-4.7:1m@max --dev sonnet-4.6@high --review codex-5.5:high@normal --subagents auto
```

Create a spec-authoring handoff when you want the configured spec model to write or refine the development artifacts:

```text
/speckit.multi-model-review.spec-handoff 001-auth-rework --spec-model codex-5.5:xhigh@normal
```

For a plan-focused pass, keep the same command and add `--plan`:

```text
/speckit.multi-model-review.spec-handoff "CSP v2 offerId alignment" --plan --spec-model codex-5.5:xhigh@normal
```

Then implement your feature as usual and package it:

```text
/speckit.multi-model-review.review-package --review-model codex-5.5:high@normal
```

Implementation still happens through your normal builder flow, such as `/speckit.implement` or direct edits. When subagent routing is enabled, the generated tasks can carry route hints so Claude Code can delegate scout, planner, worker, and local checker slices to the matching plugin subagent. `multi-model-review` records the configured dev model and refuses silent upgrade in the handoff metadata; it does not add a separate implement slash command.

The plugin writes:

```text
.cross-review/packages/<timestamp>-<slug>/
  review-package.md
  metadata.json
```

Run the reviewer in a separate terminal. Example:

```bash
PKG=.cross-review/packages/20260421-1400-auth-rework
codex exec -m codex-5.5 --file $PKG/review-package.md > $PKG/review-report.md
```

Then ingest the report:

```text
/speckit.multi-model-review.apply-review
```

If the reviewer reports `Context sufficiency: needs-full-package`, rerun:

```text
/speckit.multi-model-review.review-package --full
```

Or scope it tighter:

```text
/speckit.multi-model-review.review-package --paths src/auth,src/api
```

### Detailed option examples

Set project defaults with both model axes:

```text
/speckit.multi-model-review.cross-review init \
  --spec codex-5.5:xhigh@normal \
  --spec-heavy opus-4.7:1m@max \
  --dev sonnet-4.6@high \
  --review codex-5.5:high@normal \
  --subagents auto
```

Write a spec from the user's feature brief with high intelligence and normal speed:

```text
/speckit.multi-model-review.spec-handoff "CSP v2 offerId alignment" --spec-model codex-5.5:xhigh@normal
```

Use the heavy 1M-context spec default for a large audit:

```text
/speckit.multi-model-review.spec-handoff "DOMAINAPI full SERVER_ADDR branch audit" --heavy
```

Create a plan-focused handoff while preserving the configured implementation model:

```text
/speckit.multi-model-review.spec-handoff "CSP v2 offerId alignment" --plan --spec-model codex-5.5:xhigh@normal
```

Implementation still runs through your normal builder flow, such as `/speckit.implement` or direct edits. The configured default is `sonnet-4.6@high`, and generated metadata records `allow_silent_upgrade=false` so the workflow does not quietly move implementation to a stronger model.

Run cost-aware cross-review:

```text
/speckit.multi-model-review.review-package C-12 --review-model codex-5.5:high@normal
```

Escalate intelligence for a non-trivial design review:

```text
/speckit.multi-model-review.review-package L-30 --review-model codex-5.5:xhigh@normal
```

Use priority speed only when review turnaround is the bottleneck:

```text
/speckit.multi-model-review.review-package L-50 --review-model codex-5.5:high@priority
```

## Basic usage examples

### Codex 5.5 writes specs, Claude Sonnet 4.6 implements

Use this when you want Codex to spend the deep reasoning budget on durable Spec Kit artifacts, then keep implementation on Sonnet 4.6.

```json
{
  "builder": "claude-code",
  "model_defaults": {
    "spec": {
      "raw": "codex-5.5:xhigh@normal",
      "provider": "codex",
      "model": "codex-5.5",
      "reasoning": "xhigh",
      "speed": "normal"
    },
    "spec_heavy": {
      "raw": "opus-4.7:1m@max",
      "provider": "claude",
      "model": "claude-opus-4.7",
      "context": "1M",
      "workload": "max"
    },
    "dev": {
      "raw": "sonnet-4.6@high",
      "provider": "claude",
      "model": "claude-sonnet-4.6",
      "workload": "high",
      "allow_silent_upgrade": false
    },
    "review": {
      "raw": "codex-5.5:high@normal",
      "provider": "codex",
      "model": "codex-5.5",
      "reasoning": "high",
      "speed": "normal"
    },
    "subagent_fast": {
      "raw": "haiku-4.5@normal",
      "provider": "claude",
      "model": "claude-haiku-4.5",
      "workload": "normal"
    }
  },
  "spec_author_model": "codex-5.5",
  "spec_author_options": {
    "intelligence": "very-high",
    "reasoning": "xhigh",
    "speed": "normal"
  },
  "implementation_model": "claude-sonnet-4.6",
  "implementation_options": {
    "workload": "high",
    "allow_silent_upgrade": false
  },
  "review_model": "codex-5.5",
  "review_options": {
    "intelligence": "high",
    "reasoning": "high",
    "speed": "normal"
  },
  "reviewer": "codex-auto",
  "base_ref": "main",
  "package_profile": "compact",
  "subagent_routing": {
    "mode": "auto",
    "policy": "balanced",
    "agents": {
      "scout": { "agent": "mmr-context-scout", "model_key": "subagent_fast", "claude_code_model": "haiku" },
      "worker": { "agent": "mmr-implementation-worker", "model_key": "dev", "claude_code_model": "sonnet" },
      "heavy_planner": { "agent": "mmr-heavy-planner", "model_key": "spec_heavy", "claude_code_model": "opus[1m]" },
      "review_checker": { "agent": "mmr-review-checker", "model_key": "dev", "claude_code_model": "sonnet" }
    }
  }
}
```

Then run:

```text
/speckit.multi-model-review.spec-handoff 001-auth-rework --spec-model codex-5.5:xhigh@normal
/speckit.multi-model-review.review-package 001-auth-rework --review-model codex-5.5:high@normal
/speckit.multi-model-review.apply-review
```

### Opus 4.7 1M writes specs, Claude Sonnet 4.6 implements

Use this when the spec pass needs the Claude 1M context option and max workload setting.

```text
/speckit.multi-model-review.spec-handoff 001-auth-rework --heavy
```

Equivalent config:

```json
{
  "builder": "claude-code",
  "model_defaults": {
    "spec_heavy": {
      "raw": "opus-4.7:1m@max",
      "provider": "claude",
      "model": "claude-opus-4.7",
      "context": "1M",
      "workload": "max"
    },
    "dev": {
      "raw": "sonnet-4.6@high",
      "provider": "claude",
      "model": "claude-sonnet-4.6",
      "workload": "high",
      "allow_silent_upgrade": false
    },
    "subagent_fast": {
      "raw": "haiku-4.5@normal",
      "provider": "claude",
      "model": "claude-haiku-4.5",
      "workload": "normal"
    }
  },
  "spec_heavy_model": "claude-opus-4.7",
  "spec_heavy_options": {
    "context": "1M",
    "workload": "max"
  },
  "implementation_model": "claude-sonnet-4.6",
  "implementation_options": {
    "workload": "high",
    "allow_silent_upgrade": false
  },
  "reviewer": "codex-auto",
  "base_ref": "main",
  "package_profile": "compact",
  "subagent_routing": {
    "mode": "auto",
    "policy": "balanced"
  }
}
```

## Commands

| Spec Kit command | Legacy Claude plugin command | Purpose |
|------------------|------------------------------|---------|
| `/speckit.multi-model-review.cross-review [init|status|models set] [--spec <model[:axis]@axis>] [--spec-heavy <model[:axis]@axis>] [--dev <model[:axis]@axis>] [--review <model[:axis]@axis>] [--subagents auto\|off] [--subagent-policy conservative\|balanced\|specialist]` | `/multi-model-review:cross-review` | configure model routing, subagent roles, and defaults |
| `/speckit.multi-model-review.spec-handoff [slug|brief] [--spec-model <model[:axis]@axis>] [--heavy] [--plan] [--subagents auto\|off] [--model <id>] [--model-option <key=value>] [--dev-model <model[:axis]@axis>] [--implementation-model <id>] [--implementation-option <key=value>]` | `/multi-model-review:spec-handoff` | export the development spec or plan handoff |
| `/speckit.multi-model-review.review-package [slug|task-id] [--review-model <model[:axis]@axis>] [--base <ref>] [--full] [--micro] [--paths <glob,...>]` | `/multi-model-review:review-package` | export the reviewer handoff |
| `/speckit.multi-model-review.apply-review [path] [--min-confidence N] [--subagents auto\|off]` | `/multi-model-review:apply-review` | ingest the report and apply fixes |

## Review report contract

All reviewer templates point at the same schema in [templates/review-report.md](templates/review-report.md).

The report now includes:

- `Context sufficiency`
- `Verdict`
- `Summary`
- `Findings`

That extra `Context sufficiency` field is what makes compact-first safe: the reviewer can explicitly ask for a fuller package instead of guessing.

## Design choices

- **No hidden external side calls.** You run external reviewers yourself; Claude Code subagents are local, explicit task delegates.
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
