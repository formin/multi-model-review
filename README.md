# multi-model-review

Multi-model code review for Spec-Driven Development.

Build with **one** model (Claude Code, Codex CLI, Gemini CLI, ...), review with a **different** one. This plugin turns [GitHub Spec Kit](https://github.com/github/spec-kit) artifacts (`spec.md`, `plan.md`, `tasks.md`) into a portable handoff surface so the two models never have to talk to each other — they just read and write the same markdown files.

> The reviewer model does not need to be connected to your current session. The handoff is a single markdown file on disk. You run the other model yourself.

## Why multi-model?

A single LLM reviewing its own output tends to rationalize rather than challenge. A *different* model reading the same spec and diff has:

- Different training data — catches issues the builder's model is blind to.
- Different calibration on severity — less confirmation bias.
- Different house style — flags idiomatic drift the builder normalizes.

Spec Kit itself is great at orchestrating **one** agent through `specify → plan → tasks → implement`. It does not prescribe a cross-model review loop. `multi-model-review` fills that gap:

- The builder model does `implement`.
- `/multi-model-review:review-package` bundles the spec, plan, tasks, diff, and project rules (`CLAUDE.md`) into a self-contained prompt for the reviewer.
- You run the reviewer model on that file.
- `/multi-model-review:apply-review` ingests the reviewer's report and walks you through fixes.

Because the bundle is just markdown, any LLM with a CLI (or an API) can play reviewer. Swap models freely — even run the same diff past two different reviewers for triangulation.

## Supported reviewers out of the box

| Reviewer      | Template                                     | Example command                                                                 |
|---------------|----------------------------------------------|---------------------------------------------------------------------------------|
| Claude        | `templates/claude-review-prompt.md`          | `claude -p "$(cat <pkg>/review-package.md)" > <pkg>/review-report.md`           |
| Codex (auto)  | `templates/codex-auto-review-prompt.md`      | **Default for Codex.** Heuristic — try MCP first, fall back to CLI on timeout.  |
| Codex (CLI)   | `templates/codex-cli-review-prompt.md`       | `codex exec --file <pkg>/review-package.md > <pkg>/log.txt 2>&1 &` + `Monitor`  |
| Codex (MCP)   | `templates/codex-mcp-review-prompt.md`       | Call the `mcp__codex__codex` tool inline (<60s budget; hardcoded timeout)       |
| Gemini        | `templates/gemini-review-prompt.md`          | `gemini --file <pkg>/review-package.md > <pkg>/review-report.md`                |

Adding a new reviewer (Qwen, Mistral, a local `llama.cpp` process, ...) is a one-file change: drop `templates/<reviewer>-review-prompt.md` and use the new ID in `config.json`.

### Codex execution modes — MCP vs CLI vs auto

The `mcp__codex__codex` MCP tool hard-codes a `-32001 timed out` error at 60 seconds, so long reviews always fail when invoked via MCP. For anything past a quick sanity check, use the CLI path. `codex-auto` is the default — it tries MCP first and falls back to CLI on timeout, so you rarely need to pick manually.

| Value (`reviewer`) | Execution method                                    | Suitable for              | Failure signal                        |
|--------------------|-----------------------------------------------------|---------------------------|---------------------------------------|
| `codex-mcp`        | `mcp__codex__codex` MCP tool (inline)               | <60s short validations    | `-32001 timed out` (hardcoded at 60s) |
| `codex-cli`        | `codex exec ... > log.txt 2>&1 &` + `Monitor` tool  | Minutes to tens of minutes | No inherent timeout                  |
| `codex-auto` (default) | Heuristic + CLI fallback when MCP times out     | When unsure               | —                                     |

Pick `codex-cli` explicitly if you already know the review will take minutes; pick `codex-mcp` if you want the round-trip to stay short and fail fast if it can't. Otherwise leave it on `codex-auto`.

## Requirements

- [Claude Code](https://claude.com/claude-code) 1.x — needed for the skill and slash commands.
- A Spec Kit–initialized project (optional but recommended) — `specs/<slug>/spec.md` is picked up automatically. Without it, the plugin falls back to diff-only mode.
- The **reviewer** model installed locally (any of):
  - [Codex CLI](https://github.com/openai/codex)
  - [Gemini CLI](https://github.com/google-gemini/gemini-cli)
  - A second Claude Code instance (`claude -p`)
  - Any other CLI-capable LLM — write a template for it.
- `git` on `PATH`.

## Install

> **Slash vs shell.** Commands that start with `/` (e.g. `/plugin install`) are typed **inside a running Claude Code session** — not at your OS shell. Commands shown in ```bash``` blocks (`git …`, `claude …`, `cp …`) are typed at your OS shell (PowerShell, bash, zsh). Mixing them up returns `'/plugin' is not recognized` in PowerShell or `command not found` elsewhere.

### As a Claude Code plugin

In a Claude Code session:

```
/plugin install multi-model-review@multi-model-review
```

Or at your OS shell (equivalent):

```bash
claude plugin install multi-model-review@multi-model-review
claude plugin list      # verify "Status: enabled"
```

Then run `/reload-plugins` inside Claude Code (or restart it). The three slash commands `/multi-model-review:cross-review`, `/multi-model-review:review-package`, and `/multi-model-review:apply-review` will appear in the picker.

### Develop against a local clone

```bash
git clone https://github.com/formin/multi-model-review.git
cd multi-model-review
claude --plugin-dir .
```

After editing plugin files, run `/reload-plugins` inside Claude Code to pick up changes.

### As a standalone skill (minimal)

If you don't want the full plugin:

```bash
mkdir -p ~/.claude/skills ~/.claude/commands
cp -r skills/multi-model-review ~/.claude/skills/
cp -r templates ~/.claude/skills/multi-model-review/
cp commands/*.md ~/.claude/commands/     # optional: /cross-review, /review-package, /apply-review (no plugin namespace)
```

The skill activates on phrases like "review with Codex", "review with Gemini", "cross-review", or when you invoke any of the commands.

### Update / reinstall

Pull the latest version of the plugin from this repo — in a Claude Code session:

```
/plugin update multi-model-review@multi-model-review
```

Or at your OS shell (outside Claude Code), the equivalent CLI subcommand:

```bash
claude plugin update multi-model-review@multi-model-review
```

Uninstall the plugin (in a Claude Code session):

```
/plugin uninstall multi-model-review@multi-model-review
```

For a clean reinstall, uninstall and then install again.

In development mode (you cloned the repo and ran `claude --plugin-dir .`), update with `git pull` at your OS shell from inside the clone — no reinstall needed. If you see `fatal: destination path 'multi-model-review' already exists` when re-cloning, delete the existing directory first or pull in place:

```bash
# Windows PowerShell (at the OS shell, NOT inside Claude Code)
Remove-Item -Recurse -Force multi-model-review
git clone https://github.com/formin/multi-model-review.git

# macOS / Linux / Git Bash
rm -rf multi-model-review
git clone https://github.com/formin/multi-model-review.git

# or update in place
cd multi-model-review && git pull
```

### Starting over from a corrupted state

If the install is partially broken (install didn't complete, you renamed the clone directory):

1. In Claude Code: `/plugin uninstall multi-model-review@multi-model-review`.
2. Exit Claude Code.
3. If `~/.claude/plugins/installed_plugins.json` still has a stale entry for this plugin, hand-edit it out.
4. Delete any local clone directory if you used `--plugin-dir`.
5. Restart Claude Code and reinstall.

## Quick start

From inside a repo with a feature branch checked out:

```
/multi-model-review:cross-review init
```

Answers: `builder = claude-code`, `reviewer = gemini-cli` (or `codex-cli`, etc.), `base ref = main`.

Build your change as you normally would (or via `/speckit.implement`), then:

```
/multi-model-review:review-package
```

The plugin writes `.cross-review/packages/<timestamp>-<slug>/review-package.md` and prints the exact command to run the reviewer. Example for Gemini:

```bash
gemini --file .cross-review/packages/20260417-1020-auth-rework/review-package.md \
  > .cross-review/packages/20260417-1020-auth-rework/review-report.md
```

When the reviewer finishes:

```
/multi-model-review:apply-review
```

Claude reads the report, filters findings by confidence, and walks you through each one.

### Multi-reviewer pass (triangulate)

Nothing stops you from running the same package through two reviewers. Use the CLI form for Codex here — the MCP tool would time out on anything non-trivial:

```bash
codex  exec --file <pkg>/review-package.md > <pkg>/review-report-codex.md
gemini --file <pkg>/review-package.md      > <pkg>/review-report-gemini.md
```

Then rename one to `review-report.md` and pass the other to `/multi-model-review:apply-review <pkg>` manually, or merge the two reports by hand. A built-in multi-reviewer merge is on the roadmap.

## Commands

Claude Code namespaces plugin commands as `/<plugin-name>:<command>`. This plugin provides three:

| Invocation                                                | Purpose                                                     |
|-----------------------------------------------------------|-------------------------------------------------------------|
| `/multi-model-review:cross-review [init\|status]`          | Init config or show current roles, base ref, and packages.  |
| `/multi-model-review:review-package [slug] [--base <ref>]` | Export the handoff bundle for the reviewer model.           |
| `/multi-model-review:apply-review [path] [--min-confidence N]` | Ingest the reviewer's report and drive remediation.     |

Full details in [docs/USAGE.md](docs/USAGE.md). A scenario-driven walkthrough (Claude builds → Codex reviews → Claude applies fixes) lives in [docs/EXAMPLE.md](docs/EXAMPLE.md).

## Directory layout

```
multi-model-review/
├── .claude-plugin/
│   ├── plugin.json                    # plugin manifest
│   └── marketplace.json               # single-plugin marketplace catalog
├── skills/multi-model-review/SKILL.md # the skill (model-invoked)
├── commands/
│   └── multi-model-review.md          # single slash command, dispatches by flag
├── templates/                         # reviewer prompts + report schema
│   ├── codex-review-prompt.md          # legacy codex-cli (0.1.0 compat)
│   ├── codex-cli-review-prompt.md      # explicit `codex exec` CLI path
│   ├── codex-mcp-review-prompt.md      # `mcp__codex__codex` tool (<60s)
│   ├── codex-auto-review-prompt.md     # MCP-first, CLI fallback (default)
│   ├── claude-review-prompt.md
│   ├── gemini-review-prompt.md
│   └── review-report.md
└── docs/USAGE.md
```

At runtime, per-project state lives in `.cross-review/` inside your target repo.

## Design choices

- **No side calls.** The skill never shells out to the reviewer model. You run it yourself. This keeps auth, cost, and rate limits in your hands and avoids coupling the plugin to any one vendor's CLI flags.
- **Markdown is the only interchange.** The review package and review report are plain markdown files. Version them, diff them, commit them if you want an audit trail.
- **Confidence-gated.** Findings with `confidence < 70` are hidden by default. The reviewer explicitly scores itself, so you don't drown in nits.
- **Preserve reviewer language.** During `/multi-model-review:apply-review`, the builder quotes the report rather than paraphrasing. Prevents one model from rewriting the other's wording into agreement.
- **Template-driven, not hard-coded.** Supporting a new reviewer model = dropping a new markdown template. No code change.

## Relationship to Spec Kit

This plugin is **complementary**, not a fork. It assumes you are already using Spec Kit for `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`. If you aren't, `multi-model-review` still works in ad-hoc mode — it just has less context to hand the reviewer.

Spec Kit's own `/speckit.analyze`, `/speckit.checklist`, and `/speckit.verify` commands are single-model checks. `multi-model-review` adds the cross-model loop.

## License

MIT. See [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. This is an early version (0.1.0) and the schema for `review-report.md` may still change. Adding templates for new reviewer models is especially appreciated.
