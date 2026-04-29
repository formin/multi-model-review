---
description: Initialize config or show status for the multi-model-review workflow, including spec-author, heavy-spec, implementation, review, and subagent model routing.
argument-hint: [init|status|models set] [--spec <model[:axis]@axis>] [--spec-heavy <model[:axis]@axis>] [--dev <model[:axis]@axis>] [--review <model[:axis]@axis>] [--subagents auto|off] [--subagent-policy conservative|balanced|specialist]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# /multi-model-review:cross-review

Configure or inspect the cross-agent Spec Kit workflow.

Arguments: `$ARGUMENTS`

## Model spec syntax

Accept detailed model specs in this grammar:

```text
<model>[:<axis-a>][@<axis-b>]
```

Examples:

- `codex-5.5:xhigh@normal`
- `codex-5.5:high@priority`
- `opus-4.7:1m@max`
- `sonnet-4.6@high`

Parsing rules:

- Preserve the original string as `raw`.
- Normalize safe aliases only:
  - `xhigh` maps to legacy display `intelligence=very-high`, but keep `reasoning=xhigh` in structured defaults.
  - `1m` maps to `1M`.
  - `opus-4.7:1m` maps to Claude Opus 4.7 with `context=1M`.
  - `sonnet-4.6` maps to `claude-sonnet-4.6`.
- Do not silently upgrade or downgrade:
  - never change `high` to `xhigh`
  - never change `normal` to `priority`
  - never change `high` workload to `max`
  - never replace the selected implementation model with a stronger model
- If a known provider receives an incompatible axis, stop and explain the valid forms.

Provider mappings:

| Provider | Model examples | Axis A | Axis B |
|----------|----------------|--------|--------|
| Codex | `codex-5.5`, `gpt-5.5` | `low`, `medium`, `high`, `xhigh`, `very-high` | `normal`, `fast`, `priority` |
| Claude | `opus-4.7`, `sonnet-4.6`, `haiku-4.5`, `claude-*` | `standard`, `1m`, `1M` context when present | `low`, `normal`, `high`, `max` workload |
| Gemini or custom | any other ID | provider-specific, preserved | provider-specific, preserved |

## Subagent model routing

This plugin ships Claude Code subagents in `agents/`:

| Subagent | Default Claude Code model | Use when |
|----------|---------------------------|----------|
| `mmr-context-scout` | `haiku` | fast read-only file discovery, rule lookup, dependency mapping |
| `mmr-implementation-worker` | `sonnet` | scoped implementation tasks, tests, and follow-up fixes |
| `mmr-heavy-planner` | `opus` | cross-cutting planning, migrations, ambiguous specs, security-sensitive design |
| `mmr-review-checker` | `sonnet` | local read-only pre-review before exporting an external review package |

When `subagent_routing.mode = "auto"`, choose a subagent from the task situation, not from a single global default:

- use `mmr-context-scout` for exploration, file lists, dependency tracing, and low-risk summaries
- use `mmr-heavy-planner` before implementation when a task spans multiple subsystems, changes architecture, has unclear requirements, touches security boundaries, or needs a migration plan
- use `mmr-implementation-worker` for focused code and test edits after the task slice is clear
- use `mmr-review-checker` after implementation slices or before `/multi-model-review:review-package`

Model-selection rules:

- Prefer models already configured in `.cross-review/config.json`.
- Route normal implementation to `model_defaults.dev`.
- Route heavy planning to `model_defaults.spec_heavy`.
- Route fast scouting to `model_defaults.subagent_fast` when present, otherwise use the plugin `haiku` default.
- Route local review checking to a Claude-compatible `model_defaults.review` only when the review model is a Claude model; otherwise keep the external reviewer model for `/multi-model-review:review-package` and use `model_defaults.dev` for the local checker.
- If a Claude Code Agent invocation supports a per-call model parameter, pass the selected model for that invocation.
- If per-call model selection is not available, materialize project-local overrides in `.claude/agents/` with the selected `model` and `effort`; project agents override plugin agents.
- If the selected model provider is not supported by Claude Code subagent frontmatter, do not write that provider ID into a Claude Code agent. Keep it in handoff metadata and use the existing CLI or portable markdown path.
- Do not silently upgrade the configured implementation model. Subagent routing may choose a different configured role model for a different situation, but it must record that choice and why.

## Behavior

No argument or `status`:

- Read `.cross-review/config.json`
- Report:
  - builder
  - spec author model and options
  - heavy spec model and options, when present
  - implementation model and options
  - review model and options, when present
  - reviewer execution surface
  - subagent routing mode, policy, and role-to-model selections
  - base ref
  - default package profile
  - most recent package directory and whether it was `micro`, `compact`, or `full`
- If config is missing, prompt the user to run `/multi-model-review:cross-review init`
- If config exists but does not contain model routing keys, report them as missing and recommend rerunning `init` or adding:
  - `model_defaults`
  - `spec_author_model`
  - `spec_author_options`
  - `implementation_model`
  - `implementation_options`
  - `subagent_routing`

`models set` or `init` with model flags:

Use this when the user provides all routing details in one command instead of answering prompts:

```text
/multi-model-review:cross-review init \
    --spec codex-5.5:xhigh@normal \
    --spec-heavy opus-4.7:1m@max \
    --dev sonnet-4.6@high \
    --review codex-5.5:high@normal \
    --subagents auto
```

`models set` is accepted as an alias inside this existing command surface:

```text
/multi-model-review:cross-review models set --spec codex-5.5:xhigh@normal --spec-heavy opus-4.7:1m@max --dev sonnet-4.6@high --review codex-5.5:high@normal --subagents auto
```

1. Parse `--spec`, `--spec-heavy`, `--dev`, and `--review` using the model spec syntax above.
2. Optional flags:
   - `--builder <id>`
   - `--base <ref>`
   - `--package-profile micro|compact|full`
   - `--spec-dir <path>`
   - `--subagents auto|off`
   - `--subagent-policy conservative|balanced|specialist`
   - `--subagent-fast <model[:axis]@axis>`
   - `--no-agent-files`
3. If config is missing, require all four model flags.
4. If config exists, update only provided flags and preserve the rest.
5. Store structured defaults under `model_defaults`.
6. Also update the legacy top-level keys used by existing commands:
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
7. Set `model_defaults.dev.allow_silent_upgrade = false` unless the user explicitly passes `--allow-dev-upgrade`.
8. If `--subagent-fast` is present, parse it into `model_defaults.subagent_fast`; otherwise default it to `haiku-4.5@normal` when `subagents` is enabled.
9. Resolve `subagent_routing`:
   - `mode`: `auto` unless the user passes `--subagents off`
   - `policy`: default `balanced`
   - `agents.scout`: `mmr-context-scout`, model key `subagent_fast`
   - `agents.worker`: `mmr-implementation-worker`, model key `dev`
   - `agents.heavy_planner`: `mmr-heavy-planner`, model key `spec_heavy`
   - `agents.review_checker`: `mmr-review-checker`, model key `review` only for Claude-compatible review models, otherwise `dev`
10. If subagents are enabled and `--no-agent-files` is not present, write project-local `.claude/agents/mmr-*.md` overrides only when the resolved model is Claude Code compatible. Use Claude Code model aliases when possible:
   - Opus models -> `opus`, or `opus[1m]` when context is `1M`
   - Sonnet models -> `sonnet`, or `sonnet[1m]` when context is `1M`
   - Haiku models -> `haiku`
   - custom Claude-compatible full IDs -> the full ID
   - non-Claude providers -> skip materialization and record a warning in `subagent_routing.notes`
   - use the matching plugin `agents/mmr-*.md` body and change only `model`, `effort`, and explanatory routing comments as needed
   - if a project-local `.claude/agents/mmr-*.md` already exists and does not look generated by this plugin, ask before overwriting it
11. Write `.cross-review/config.json`.
12. Add `.cross-review/` to `.gitignore` unless the user explicitly wants to commit it.
13. Print the selected routing in a short table, including subagent role, agent name, selected model key, and materialized Claude Code model when applicable.

Interactive `init` without model flags:

1. Ask which model should author or refine the development spec documents.
   - recommended: `codex-5.5:xhigh@normal`
   - alternative: `opus-4.7:1m@max`
   - custom ID

2. Ask whether to use the default options for that model.
   - for Codex `codex-5.5` or `gpt-5.5`, default options are:
     - `reasoning = xhigh`
     - `intelligence = very-high`
     - `speed = normal`
   - for `opus-4.7`, default options are:
     - `context = 1M`
     - `workload = max`
   - accepted Codex values:
     - `reasoning = low | medium | high | xhigh`
     - `intelligence = low | medium | high | very-high`
     - `speed = normal | fast | priority`
   - accepted Claude values:
     - `context = standard | 1M`
     - `workload = low | normal | high | max`
   - for custom IDs, accept key/value pairs and store them under `spec_author_options`
   - derive `spec_author_profile` from the options for display, but treat `spec_author_options` as the legacy source of truth

3. Ask which heavy spec default should be used for large audits.
   - recommended: `opus-4.7:1m@max`
   - store it in `model_defaults.spec_heavy`, `spec_heavy_model`, `spec_heavy_options`, and `spec_heavy_profile`

4. Ask which agent is the implementation builder surface.
   - `claude-code`
   - `codex-cli`
   - `codex-mcp`
   - `codex-auto`
   - `gemini-cli`
   - custom ID

5. Ask which model should do the actual development implementation.
   - default `sonnet-4.6@high`
   - Claude model choices:
     - `opus-4.7:1m@max`
     - `sonnet-4.6@high`
     - `haiku-4.5@normal`
   - explain that this is separate from the builder surface and is optimized for token-heavy implementation work
   - custom ID

6. Ask which options should be used for the implementation model.
   - default for Claude implementation models:
     - `workload = high`
   - if the model is `claude-opus-4.7` with `context=1M`, store `context = 1M`
   - accept key/value pairs and store them under `implementation_options`
   - default to `allow_silent_upgrade = false`

7. Ask which review model should be used.
   - default `codex-5.5:high@normal`
   - use `codex-5.5:xhigh@normal` only for non-trivial design reviews
   - use `codex-5.5:high@priority` only when review turnaround is the bottleneck

8. Ask which agent is the reviewer.
   - warn, but do not block, if builder and reviewer are the same
   - for Codex, explain:
     - `codex-auto`: default
     - `codex-cli`: long-running reviews
     - `codex-mcp`: short MCP checks only

9. Ask whether to enable automatic subagent routing.
   - default `auto`
   - choose `off` only when the user wants all work to stay in the main conversation
   - explain that the plugin ships `mmr-context-scout`, `mmr-implementation-worker`, `mmr-heavy-planner`, and `mmr-review-checker`

10. Ask which subagent selection policy to use.
   - `conservative`: delegate only obvious read-only scout or review-checker tasks
   - `balanced`: default; delegate clear scout, planner, worker, and review-checker slices
   - `specialist`: prefer specialized subagents whenever a task has a clear route

11. Ask which fast scout model should be used.
   - default `haiku-4.5@normal`
   - store it in `model_defaults.subagent_fast`
   - use it for `mmr-context-scout` unless the user chooses a different Claude-compatible model

12. Ask whether to materialize project-local subagent overrides.
   - default yes when `builder = claude-code` and `subagents = auto`
   - write `.claude/agents/mmr-*.md` files with resolved Claude Code model aliases
   - skip any role whose selected model is not Claude Code compatible and record a config note

13. Ask for base ref.
   - default `main`

14. Ask for default package profile.
   - `compact` (recommended)
   - `full`
   - `micro` (only appropriate for MCP smoke checks)
   - if reviewer is `codex-mcp`, force `micro`

15. Ask for the Spec Kit feature directory.
   - offer `specs/*/` choices when available
   - allow ad-hoc mode

16. Write `.cross-review/config.json`.

17. Add `.cross-review/` to `.gitignore` unless the user explicitly wants to commit it.

Example config:

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
  "spec_author_profile": "intelligence=very-high, reasoning=xhigh, speed=normal",
  "spec_heavy_model": "claude-opus-4.7",
  "spec_heavy_options": {
    "context": "1M",
    "workload": "max"
  },
  "spec_heavy_profile": "context=1M, workload=max",
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
  "review_profile": "intelligence=high, reasoning=high, speed=normal",
  "reviewer": "codex-auto",
  "base_ref": "main",
  "spec_dir": "specs/001-auth-rework",
  "package_profile": "compact",
  "subagent_routing": {
    "mode": "auto",
    "policy": "balanced",
    "agents": {
      "scout": {
        "agent": "mmr-context-scout",
        "model_key": "subagent_fast",
        "claude_code_model": "haiku",
        "effort": "medium"
      },
      "worker": {
        "agent": "mmr-implementation-worker",
        "model_key": "dev",
        "claude_code_model": "sonnet",
        "effort": "high"
      },
      "heavy_planner": {
        "agent": "mmr-heavy-planner",
        "model_key": "spec_heavy",
        "claude_code_model": "opus[1m]",
        "effort": "xhigh"
      },
      "review_checker": {
        "agent": "mmr-review-checker",
        "model_key": "dev",
        "claude_code_model": "sonnet",
        "effort": "high",
        "note": "external review model is codex, so local checker uses dev"
      }
    },
    "selection_rules": [
      "scout: read-only discovery and dependency mapping",
      "heavy_planner: cross-cutting, ambiguous, migration, or security-sensitive planning",
      "worker: scoped code and test edits after the slice is clear",
      "review_checker: local preflight before external review packaging"
    ]
  }
}
```

## Output style

Keep status output short and scannable.

## Refer to

- `/multi-model-review:review-package`
- `/multi-model-review:spec-handoff`
- `/multi-model-review:apply-review`
- `skills/multi-model-review/SKILL.md`
