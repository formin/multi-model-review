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
- `/review-package` bundles the spec, plan, tasks, diff, and project rules (`CLAUDE.md`) into a self-contained prompt for the reviewer.
- You run the reviewer model on that file.
- `/apply-review` ingests the reviewer's report and walks you through fixes.

Because the bundle is just markdown, any LLM with a CLI (or an API) can play reviewer. Swap models freely — even run the same diff past two different reviewers for triangulation.

## Supported reviewers out of the box

| Reviewer   | Template                                   | Example command                                                                 |
|------------|--------------------------------------------|---------------------------------------------------------------------------------|
| Claude     | `templates/claude-review-prompt.md`        | `claude -p "$(cat <pkg>/review-package.md)" > <pkg>/review-report.md`          |
| Codex      | `templates/codex-review-prompt.md`         | `codex exec --file <pkg>/review-package.md > <pkg>/review-report.md`           |
| Gemini     | `templates/gemini-review-prompt.md`        | `gemini --file <pkg>/review-package.md > <pkg>/review-report.md`               |

Adding a new reviewer (Qwen, Mistral, a local `llama.cpp` process, ...) is a one-file change: drop `templates/<reviewer>-review-prompt.md` and use the new ID in `config.json`.

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

### As a Claude Code plugin

```bash
git clone https://github.com/formin/multi-model-review.git \
  ~/.claude/plugins/local/multi-model-review
```

Then, in Claude Code:

```
/plugin
```

Point the plugin UI at the cloned path, or add a marketplace entry referencing this repo.

### As a standalone skill (minimal)

If you don't want the full plugin:

```bash
mkdir -p ~/.claude/skills
cp -r skills/cross-review ~/.claude/skills/
cp -r commands/* ~/.claude/commands/     # optional: slash commands
cp -r templates ~/.claude/skills/cross-review/
```

The skill activates on phrases like "review with Codex", "review with Gemini", "cross-review", or when you type `/cross-review`.

## Quick start

From inside a repo with a feature branch checked out:

```
/cross-review init
```

Answers: `builder = claude-code`, `reviewer = gemini-cli` (or `codex-cli`, etc.), `base ref = main`.

Build your change as you normally would (or via `/speckit.implement`), then:

```
/review-package
```

The plugin writes `.cross-review/packages/<timestamp>-<slug>/review-package.md` and prints the exact command to run the reviewer. Example for Gemini:

```bash
gemini --file .cross-review/packages/20260417-1020-auth-rework/review-package.md \
  > .cross-review/packages/20260417-1020-auth-rework/review-report.md
```

When the reviewer finishes:

```
/apply-review
```

Claude reads the report, filters findings by confidence, and walks you through each one.

### Multi-reviewer pass (triangulate)

Nothing stops you from running the same package through two reviewers:

```bash
codex  exec --file <pkg>/review-package.md > <pkg>/review-report-codex.md
gemini --file <pkg>/review-package.md       > <pkg>/review-report-gemini.md
```

Then rename one to `review-report.md` and pass the other to `/apply-review <pkg>` manually, or merge the two reports by hand. A built-in multi-reviewer merge is on the roadmap.

## Commands

| Command           | Purpose                                                     |
|-------------------|-------------------------------------------------------------|
| `/cross-review`   | Status / init / route. Entry point.                         |
| `/review-package` | Export the handoff bundle for the reviewer model.           |
| `/apply-review`   | Ingest the reviewer's report and drive remediation.         |

Full details in [docs/USAGE.md](docs/USAGE.md).

## Directory layout

```
multi-model-review/
├── .claude-plugin/plugin.json         # plugin manifest
├── skills/cross-review/SKILL.md       # the skill (model-invoked)
├── commands/                          # slash commands (user-invoked)
│   ├── cross-review.md
│   ├── review-package.md
│   └── apply-review.md
├── templates/                         # reviewer prompts + report schema
│   ├── codex-review-prompt.md
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
- **Preserve reviewer language.** During `/apply-review`, the builder quotes the report rather than paraphrasing. Prevents one model from rewriting the other's wording into agreement.
- **Template-driven, not hard-coded.** Supporting a new reviewer model = dropping a new markdown template. No code change.

## Relationship to Spec Kit

This plugin is **complementary**, not a fork. It assumes you are already using Spec Kit for `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`. If you aren't, `multi-model-review` still works in ad-hoc mode — it just has less context to hand the reviewer.

Spec Kit's own `/speckit.analyze`, `/speckit.checklist`, and `/speckit.verify` commands are single-model checks. `multi-model-review` adds the cross-model loop.

## License

MIT. See [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. This is an early version (0.1.0) and the schema for `review-report.md` may still change. Adding templates for new reviewer models is especially appreciated.
