---
name: cross-review
description: This skill should be used when the user asks to run a multi-model / cross-agent code review ("build with Claude, review with Codex", "have Gemini review this", "get a second opinion from another model", "cross-review"), wants to bridge spec-kit artifacts between Claude Code, Codex CLI, Gemini CLI, or any other agent, or invokes `/cross-review`, `/review-package`, or `/apply-review`. It coordinates the "builder vs reviewer" handoff using portable spec/plan/tasks/diff artifacts — never calls the other agent directly; instead produces a review-package file the user feeds to the chosen reviewer model, and ingests the review report that comes back. Reviewer selection is template-driven, so any agent with a matching `templates/<agent>-review-prompt.md` works.
version: 0.1.0
license: MIT
---

# Multi-Model Spec Review

Coordinate a "builder agent vs reviewer agent" workflow across different LLMs (Claude, Codex, Gemini, Qwen, ...) on top of GitHub Spec Kit artifacts. The skill does not invoke the other agent — it produces a portable handoff package, and ingests the review report the user brings back.

## When to activate

Activate when the user:

- Asks to "review with Codex" / "review with Gemini" / "review with Claude" / "cross-review" / "second opinion from another model"
- Wants to split build and review across different LLMs (Claude Code, Codex CLI, Gemini CLI, etc.)
- Mentions spec-kit artifacts (`spec.md`, `plan.md`, `tasks.md`) in a review context
- Invokes `/multi-model-review:cross-review`, `/multi-model-review:review-package`, or `/multi-model-review:apply-review`

## Mental model

```
                 spec.md   plan.md   tasks.md   diff
                    \        |         /         /
                     \       |        /         /
                      +------v-------+---------+
                      |  review-package.md     |  <-- portable, model-agnostic
                      +-----------+------------+
                                  |
                 user runs the OTHER model on this file
                                  |
                      +-----------v------------+
                      |  review-report.md      |  <-- reviewer writes this
                      +-----------+------------+
                                  |
                     builder ingests & addresses findings
```

The skill owns two directions of the pipe:

1. **Export** — gather spec-kit artifacts + diff + context into a single review-package file.
2. **Ingest** — parse the review-report file the other model produced and drive remediation.

## Directory convention

All handoff files live under `.cross-review/` at the repo root:

```
.cross-review/
  config.json                    # builder/reviewer roles, base ref
  packages/<timestamp>-<slug>/
    review-package.md            # what the reviewer reads
    review-report.md             # what the reviewer writes back
    metadata.json                # base ref, head ref, builder, reviewer
```

If `.cross-review/config.json` is absent, create it on first use. Ask the user:

- Which agent is the **builder** (any of: `claude-code`, `codex-cli`, `gemini-cli`, or any identifier the user has a template for)?
- Which agent is the **reviewer** (a *different* one — the whole point is a different model's perspective)?
- What is the **base ref** to diff against (default: `main`)?
- Is there a spec-kit project initialized (look for `specs/*/spec.md`)?

Warn (don't block) if builder == reviewer: the value of this workflow is cross-model perspective.

## Export flow (`/multi-model-review:review-package`)

Build a self-contained markdown file the reviewer model can read with zero other context.

1. Resolve the spec-kit feature directory:
   - If user gave a slug, use `specs/<slug>/`
   - Else pick the most recently modified `specs/*/` directory
   - If no spec-kit project exists, fall back to "ad-hoc" mode: describe the change from the diff alone

2. Collect inputs:
   - `specs/<slug>/spec.md` (what)
   - `specs/<slug>/plan.md` (how)
   - `specs/<slug>/tasks.md` (steps)
   - `git diff <base>...HEAD` (the change under review)
   - `git log <base>...HEAD --oneline` (commit trail)
   - Root `CLAUDE.md` and any `CLAUDE.md` in directories the diff touched (the builder's rules — the reviewer should know them)

3. Assemble `review-package.md` using the template at `templates/<reviewer>-review-prompt.md` (e.g. `codex-review-prompt.md`, `claude-review-prompt.md`, `gemini-review-prompt.md`). If no template matches the configured reviewer, prompt the user to drop in a new template file — the plugin is extensible by file convention, no code change needed. The template wraps the collected inputs in a reviewer-role prompt that instructs the reviewer to:
   - Check spec-to-code alignment (does the diff actually implement what `spec.md` and `tasks.md` say?)
   - Flag bugs, security issues, and CLAUDE.md violations
   - Score each finding 0–100 for confidence
   - Output a `review-report.md` file in the exact format in `templates/review-report.md`

4. Write outputs under `.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/`:
   - `review-package.md`
   - `metadata.json` with base/head commits + roles

5. Print the reviewer command the user should run next. Examples:
   - Codex CLI: `codex exec --file .cross-review/packages/<pkg>/review-package.md`
   - Claude Code: `claude -p "$(cat .cross-review/packages/<pkg>/review-package.md)"`
   - Gemini CLI: `gemini --file .cross-review/packages/<pkg>/review-package.md`

**Do not** try to invoke the other model yourself. The handoff is user-driven.

## Ingest flow (`/multi-model-review:apply-review`)

The user has run the reviewer model and saved its output as `review-report.md` in the package directory (or pasted its content).

1. Locate the latest package dir under `.cross-review/packages/` unless the user specified one.
2. Read `review-report.md`. It should follow the schema in `templates/review-report.md`:
   - Findings list, each with: severity, confidence (0–100), file:line, description, suggested fix
   - Optional overall verdict
3. Filter: by default drop findings with confidence < 70 unless the user asks for all.
4. Present findings as a prioritized checklist. Do not auto-edit code.
5. For each accepted finding, walk the user through the fix — read the file, propose an edit, apply it after confirmation.
6. After fixes land, remind the user they can run `/multi-model-review:review-package` again for a second pass, optionally with a *different* reviewer model to triangulate.

## Role swap & adding new reviewers

The same skill handles every direction. The only things that change between "Claude builds / Codex reviews", "Codex builds / Gemini reviews", "Gemini builds / Claude reviews", etc. are:

- Which model is active when `/multi-model-review:review-package` runs (the builder)
- Which template is used to wrap the package — lookup is `templates/<reviewer>-review-prompt.md`
- Which model is active when `/multi-model-review:apply-review` runs (the builder, again)

Record the roles in `.cross-review/config.json` so subsequent runs don't re-ask.

**Adding a new reviewer** (e.g. Qwen, Mistral, a local model): drop a new file at `templates/<new-reviewer>-review-prompt.md` following the shape of the existing ones, then use that reviewer ID in config. No code change needed.

## Important constraints

- **No side calls to external models.** The skill must not shell out to Codex, Gemini, or invoke Claude via the API — the user runs the other model.
- **Artifacts only.** All state lives in `.cross-review/` and the spec-kit `specs/` tree. Nothing else.
- **Never auto-apply a finding with confidence < 70**, even if the user has approved the whole batch — re-confirm for each low-confidence item.
- **Never escalate severity.** If the reviewer marked a finding "minor", do not re-rank it to "critical" during ingest.
- **Preserve the reviewer's language.** When summarizing findings to the user, quote the report; don't paraphrase the reviewer into agreement. Different models have different calibration — their own wording matters.

## Related commands

- `/multi-model-review:cross-review [init|status]` — interactive config / status
- `/multi-model-review:review-package [slug] [--base <ref>]` — export flow
- `/multi-model-review:apply-review [package-dir] [--min-confidence N]` — ingest flow

## Related files

- `templates/codex-review-prompt.md` — wraps artifacts for Codex as reviewer
- `templates/claude-review-prompt.md` — wraps artifacts for Claude as reviewer
- `templates/gemini-review-prompt.md` — wraps artifacts for Gemini as reviewer
- `templates/review-report.md` — the schema every reviewer must follow (model-agnostic)
