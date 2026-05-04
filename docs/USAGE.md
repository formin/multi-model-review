# Usage guide

Step-by-step walkthrough for `multi-model-review`.

## 1. Prerequisites

- Claude Code installed and authenticated
- `git` on `PATH`
- at least one reviewer model installed locally
- Spec Kit initialized with `specify init` when using the Spec Kit extension install path

## Command names

When installed through Spec Kit, use the `speckit.multi-model-review.*` command namespace:

| Spec Kit command | Legacy Claude plugin command |
|------------------|------------------------------|
| `/speckit.multi-model-review.cross-review` | `/multi-model-review:cross-review` |
| `/speckit.multi-model-review.spec-handoff` | `/multi-model-review:spec-handoff` |
| `/speckit.multi-model-review.review-package` | `/multi-model-review:review-package` |
| `/speckit.multi-model-review.apply-review` | `/multi-model-review:apply-review` |

Recommended model routing:

| Stage | Model |
|-------|-------|
| development spec author | `codex-5.5:xhigh@normal` |
| heavy spec author | `opus-4.7:1m@max` |
| actual development implementation | `sonnet-4.6@high` |
| review | `codex-5.5:high@normal` |
| fast subagent scout | `haiku-4.5@normal` |
| subagent routing | `auto` |

Detailed model specs use:

```text
<model>[:<axis-a>][@<axis-b>]
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

Supported reviewer examples:

- Codex CLI
- Codex MCP
- Gemini CLI
- another Claude session

Subagent routing is optional but recommended when implementation work will be split:

| Route | Agent | Default model | Use for |
|-------|-------|---------------|---------|
| `scout` | `mmr-context-scout` | `haiku` | read-only discovery and summaries |
| `heavy-planner` | `mmr-heavy-planner` | `opus` | cross-cutting design, migrations, security-sensitive plans |
| `worker` | `mmr-implementation-worker` | `sonnet` | scoped implementation and tests |
| `review-checker` | `mmr-review-checker` | `sonnet` | local read-only preflight |

## 2. Initialize the workflow

Inside the target repo:

```text
/multi-model-review:cross-review init \
  --spec codex-5.5:xhigh@normal \
  --spec-heavy opus-4.7:1m@max \
  --dev sonnet-4.6@high \
  --review codex-5.5:high@normal \
  --subagents auto
```

Equivalent update form:

```text
/multi-model-review:cross-review models set --spec codex-5.5:xhigh@normal --spec-heavy opus-4.7:1m@max --dev sonnet-4.6@high --review codex-5.5:high@normal --subagents auto
```

Typical stored values:

| Prompt | Example |
|--------|---------|
| builder | `claude-code` |
| spec default | `codex-5.5:xhigh@normal` |
| heavy spec default | `opus-4.7:1m@max` |
| implementation default | `sonnet-4.6@high` |
| review default | `codex-5.5:high@normal` |
| subagent routing | `auto`, policy `balanced` |
| reviewer | `codex-auto` |
| base ref | `main` |
| package profile | `compact` |
| spec dir | `specs/001-auth-rework` |

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
  "spec_dir": "specs/001-auth-rework",
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

## 3. Prepare development specs

When you want a stronger or longer-context model to write or refine the Spec Kit artifacts before development, create a spec handoff:

```text
/multi-model-review:spec-handoff 001-auth-rework
```

Override model routing for a single handoff when needed:

```text
/multi-model-review:spec-handoff 001-auth-rework --spec-model codex-5.5:xhigh@normal
/multi-model-review:spec-handoff 001-auth-rework --spec-model opus-4.7:1m@max
/multi-model-review:spec-handoff 001-auth-rework --heavy
/multi-model-review:spec-handoff "CSP v2 offerId alignment" --plan --spec-model codex-5.5:xhigh@normal
/multi-model-review:spec-handoff "Add magic-link auth" --dev-model sonnet-4.6@high
```

`model_defaults` is the structured source of truth. Legacy `spec_author_options` and `spec_author_profile` still appear in generated prompts, for example `intelligence=very-high, reasoning=xhigh, speed=normal`.

The command writes:

```text
.cross-review/spec-handoffs/<YYYYMMDD-HHMM>-<slug>/
  spec-authoring-prompt.md
  metadata.json
```

Run the selected spec author model yourself and save its output as `spec-output.md`. Apply the returned file blocks to `spec.md`, `plan.md`, and `tasks.md` after reviewing them.

## 4. Build the feature

Implement the feature however you normally work:

- Spec Kit path: `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`
- ad-hoc path: edit code directly

Use the configured `implementation_model` and `implementation_options` for the token-heavy implementation pass. The default is `sonnet-4.6@high`, stored as `claude-sonnet-4.6` with `workload=high`.

The default implementation routing disables silent upgrade. If the configured dev model is `sonnet-4.6@high`, do not silently switch to Opus or a higher workload just because the task looks hard.

When subagent routing is enabled, let the route hints in `tasks.md` guide delegation:

- `[route:scout]` -> use `mmr-context-scout`
- `[route:heavy-planner]` -> use `mmr-heavy-planner`
- `[route:worker]` -> use `mmr-implementation-worker`
- `[route:review-checker]` -> use `mmr-review-checker`

If a configured role model is not usable as a Claude Code subagent model, keep that model in the handoff or CLI review path and use the recorded fallback for the local subagent.

Commit the branch when possible. The reviewer can inspect uncommitted changes in the diff, but the commit trail is more useful when commits exist.

## 5. Export the review package

```text
/multi-model-review:review-package
```

Optional variants:

```text
/multi-model-review:review-package --full
/multi-model-review:review-package --paths src/auth,src/api
/multi-model-review:review-package 001-auth-rework --base release/2026-q2
/multi-model-review:review-package C-12 --review-model codex-5.5:high@normal
/multi-model-review:review-package L-30 --review-model codex-5.5:xhigh@normal
/multi-model-review:review-package L-50 --review-model codex-5.5:high@priority
```

What the command now does:

1. resolves the feature slug and base ref
2. resolves the review model, including any `--review-model` override
3. profiles the diff
4. builds a compact package by default
5. writes:

```text
.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/
  review-package.md
  metadata.json
```

The compact package contains summaries, a diff manifest, focused excerpts, and omission notes. It does not dump every raw artifact by default.

## 6. Run the reviewer yourself

Examples:

```bash
PKG=.cross-review/packages/20260421-1400-auth-rework

codex exec -m codex-5.5 --file $PKG/review-package.md > $PKG/review-report.md
gemini --file $PKG/review-package.md > $PKG/review-report.md
claude -p "$(cat $PKG/review-package.md)" > $PKG/review-report.md
```

For `codex-mcp`, the skill may use the MCP path inline for very small packages. Anything longer should move to the CLI path.

## 7. Review report structure

The reviewer writes `review-report.md` using [templates/review-report.md](../templates/review-report.md).

Important fields:

- `Context sufficiency`
- `Verdict`
- `Summary`
- `Findings`

`Context sufficiency` is the compact-first escape hatch:

- `sufficient`: proceed normally
- `limited-but-actionable`: proceed, but note the scope warning
- `needs-full-package`: rerun packaging with more context

## 8. Ingest the report

```text
/multi-model-review:apply-review
```

Optional:

```text
/multi-model-review:apply-review --min-confidence 85
/multi-model-review:apply-review .cross-review/packages/20260421-1400-auth-rework
/multi-model-review:apply-review --subagents auto
```

Default behavior:

1. parse the report
2. stop early if the report says `needs-full-package`
3. drop findings with confidence below 70
4. present a checklist
5. apply accepted findings one at a time
6. when subagent routing is enabled, use the implementation worker and local review checker for accepted path-local fixes

## 9. When to rerun with `--full`

Use `--full` when:

- the reviewer explicitly asks for it
- the diff is too cross-cutting for focused excerpts
- rule interpretation depends on large omitted appendices
- the review is security-sensitive and you want maximum raw context

Do not default to `--full` for every review. The compact package is the normal path.

## 10. When to use `--paths`

Use `--paths` when the full branch is too large but the issue is local:

- `src/auth`
- `src/payments`
- `db/migrations`

This often gives better signal than switching blindly to a huge `--full` package.

## 11. RTK-inspired behavior

The package strategy borrows the same ideas that RTK applies to shell output:

- filter noise
- group related items
- truncate repetitive output
- deduplicate repeated structure

See [TOKEN_EFFICIENCY.md](TOKEN_EFFICIENCY.md) for the detailed mapping.

## 12. Troubleshooting

### No `specs/<slug>` directory found

The command falls back to ad-hoc diff-only mode.

### Reviewer says `needs-full-package`

Rerun:

```text
/multi-model-review:review-package --full
```

Or narrow the package:

```text
/multi-model-review:review-package --paths <subset>
```

### Codex MCP times out

Use `codex-auto` or `codex-cli`. MCP is only for very small packages.

### Findings feel noisy

Raise the confidence floor:

```text
/multi-model-review:apply-review --min-confidence 85
```

### Findings feel sparse

Lower the floor or ask for a fuller package.
