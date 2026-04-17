# Usage guide

Step-by-step walkthrough for `multi-model-review`.

## 0. Prerequisites

- Claude Code installed and authenticated.
- `git` on `PATH`. Working directory is a git repo.
- The **reviewer** model installed — whichever one(s) you plan to use:
  - Codex CLI: `codex --version` should work.
  - Gemini CLI: `gemini --version` should work.
  - Claude Code as reviewer: you'll use `claude -p` in a separate terminal.
  - Any other CLI-capable LLM (Qwen, local Llama, ...) — you'll need a matching template under `templates/`.
- (Recommended) Spec Kit initialized in the repo, so `specs/<slug>/spec.md` exists.

## 1. Install the plugin

See [README.md](../README.md#install). The shortest path — inside any Claude Code session:

```
/plugin install spec-cross-review@multi-model-review
```

Or at your OS shell (equivalent):

```bash
claude plugin install spec-cross-review@multi-model-review
claude plugin list          # verify "Status: enabled"
```

`/reload-plugins` or restart Claude Code. `/multi-model-review` should now appear in the slash-command list.

### Updating, reinstalling, starting over

> `/plugin …` commands go inside a **Claude Code session**. `claude …` and `git …` commands go at your **OS shell** (PowerShell, bash, zsh). Pasting `/plugin …` at a PowerShell prompt returns `'/plugin' is not recognized`.

- **Update** to the latest version — in Claude Code: `/plugin update spec-cross-review@multi-model-review`. Or at the shell: `claude plugin update spec-cross-review@multi-model-review`.
- **Uninstall** — in Claude Code: `/plugin uninstall spec-cross-review@multi-model-review`.
- **Dev mode (`claude --plugin-dir`)**: at the shell, `git pull` inside your clone. If `git clone` fails with `destination path already exists`, either delete the existing directory (`rm -rf multi-model-review` / `Remove-Item -Recurse -Force multi-model-review` on PowerShell) or `git pull` in place.

## 2. Initialize a project

In your target repo, from Claude Code:

```
/multi-model-review --cross-review init
```

Answer the prompts:

| Question              | Typical answer                                                             |
|-----------------------|----------------------------------------------------------------------------|
| Builder agent         | `claude-code` (you're in Claude right now, so probably this)               |
| Reviewer agent        | `codex-cli` / `gemini-cli` / `claude-code` — pick a **different** model    |
| Base ref              | `main` (or whichever branch you diff against)                              |
| Spec-kit feature dir  | e.g. `specs/001-auth-rework/` — glob shows what's available                |

A config file is written to `.cross-review/config.json`:

```json
{
  "builder": "claude-code",
  "reviewer": "gemini-cli",
  "base_ref": "main",
  "spec_dir": "specs/001-auth-rework"
}
```

`.cross-review/` is added to `.gitignore` unless you explicitly opt in to committing it.

> **Tip:** Picking `reviewer = builder` is allowed but the plugin will warn. The whole point is cross-model perspective — use a different model if at all possible.

## 3. Build your change

Nothing plugin-specific here. Implement the feature however you normally would:

- Spec Kit path: `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`.
- Ad-hoc path: just write code, commit locally.

Commit your work on a feature branch. The reviewer sees the diff against `base_ref`, so commits must be on the branch — uncommitted changes are included in the diff too, but they won't show up in `git log`.

## 4. Export the review package

```
/multi-model-review --review-package
```

Optional arguments:

- Positional slug — override which `specs/<slug>/` to use.
- `--base <ref>` — override the base ref for this run.

What it does:

1. Resolves `<slug>` (from args, or the most recently modified `specs/*/`).
2. Computes `BASE = git merge-base <base_ref> HEAD`.
3. Gathers `spec.md`, `plan.md`, `tasks.md`, the diff, the commit log, and every `CLAUDE.md` that applies.
4. Picks the right reviewer template based on `config.reviewer` — looks up `templates/<reviewer>-review-prompt.md`.
5. Writes to:

```
.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/
├── review-package.md     # the reviewer reads this
└── metadata.json         # base, head, builder, reviewer, timestamp
```

6. Prints the command to run the reviewer. For example:

```bash
# reviewer = codex-cli
codex exec --file .cross-review/packages/20260417-1020-auth-rework/review-package.md \
  > .cross-review/packages/20260417-1020-auth-rework/review-report.md

# reviewer = gemini-cli
gemini --file .cross-review/packages/20260417-1020-auth-rework/review-package.md \
  > .cross-review/packages/20260417-1020-auth-rework/review-report.md

# reviewer = claude-code
claude -p "$(cat .cross-review/packages/20260417-1020-auth-rework/review-package.md)" \
  > .cross-review/packages/20260417-1020-auth-rework/review-report.md
```

### Safety checks

The export refuses to run if:

- The diff contains likely secrets (`AKIA…`, `-----BEGIN`, `ghp_…`, `sk-…`). Scrub and retry.
- The package would exceed ~200 KB. Split by path and re-run with a narrower scope.

## 5. Run the reviewer model

**You** do this, not the plugin. Open a separate terminal. Use one of the commands printed in step 4, matching your configured reviewer.

The template inside the package tells the reviewer to produce output in the schema defined in [templates/review-report.md](../templates/review-report.md). Every supported reviewer template points at the same schema, so the downstream parsing in `/multi-model-review --apply-review` is model-agnostic.

### Multi-reviewer triangulation

You can send the same package to more than one reviewer to get independent signals:

```bash
codex  exec --file <pkg>/review-package.md > <pkg>/review-report-codex.md
gemini --file <pkg>/review-package.md       > <pkg>/review-report-gemini.md
claude -p "$(cat <pkg>/review-package.md)"  > <pkg>/review-report-claude.md
```

Findings that appear in 2+ reports are almost certainly real. Findings unique to one reviewer are where cross-model value lives — but treat them with healthy skepticism.

To ingest one of them, rename it to `review-report.md` before running `/multi-model-review --apply-review` (a built-in multi-report merge is on the roadmap).

## 6. Ingest the review

Back in your Claude Code session in the project:

```
/multi-model-review --apply-review
```

Optional arguments:

- Positional path to a package dir — otherwise the latest one with a `review-report.md` is used.
- `--min-confidence N` — default 70. Use `--min-confidence 0` to see every finding.

What happens:

1. The report is parsed. Each finding has `severity`, `confidence`, `location`, `summary`, `detail`, `suggested_fix`.
2. Low-confidence findings are dropped.
3. Remaining findings are presented as a numbered checklist.
4. For each finding you accept, Claude reads the target file, proposes an `Edit`, and applies it after confirmation.
5. State is tracked in `.cross-review/packages/<pkg>/review-state.json`, so re-runs know which findings were already applied.

### Guardrails during ingest

- `severity: critical` is never auto-applied in batch mode — always explicit confirm.
- Two findings on the same line are applied sequentially with a re-read between.
- The builder model quotes the reviewer's wording instead of paraphrasing. Different models calibrate differently — keeping the reviewer's language preserves that signal.

## 7. Iterate

After applying fixes, commit and run `/multi-model-review --review-package` again. Each run creates a new timestamped package, so you keep an audit trail of review rounds. Consider rotating reviewers between rounds — e.g. round 1 reviewed by Codex, round 2 reviewed by Gemini — to surface different classes of issues.

## Swapping roles

To flip to e.g. "Codex builds, Claude reviews":

1. Edit `.cross-review/config.json` — swap `builder` and `reviewer`.
2. Next `/multi-model-review --review-package` will use [templates/claude-review-prompt.md](../templates/claude-review-prompt.md) instead of the Codex one.
3. Build your change with Codex (outside this Claude session).
4. In Codex, ensure the feature branch is committed.
5. Come back to Claude and run `/multi-model-review --review-package` — Claude is now the reviewer, and you run it via `claude -p ...`.

Or just re-run `/multi-model-review --cross-review init` to reset interactively.

## Adding a new reviewer model

Say you want Qwen CLI as a reviewer:

1. Copy an existing template:
   ```bash
   cp templates/claude-review-prompt.md templates/qwen-review-prompt.md
   ```
2. Edit the top of `qwen-review-prompt.md` — change the reviewer name in the header. The body (instructions, schema, confidence scoring) is model-agnostic and can stay identical.
3. Set `reviewer: "qwen-cli"` in `.cross-review/config.json`.
4. Add your Qwen CLI invocation to the printout logic (for a personal setup, just memorize it — for a PR back to this repo, update the README table).

That's it. No code change, no plugin rebuild.

## Troubleshooting

### "No `specs/<slug>/` directory found"

You're in ad-hoc mode. `/multi-model-review --review-package` will still run, but the reviewer only sees the diff and `CLAUDE.md`, not a spec. Either initialize Spec Kit or write a one-page `specs/<slug>/spec.md` by hand.

### "Package would contain likely secrets"

Grep your diff for the patterns in the error message and remove them (rotate keys first if committed).

### "review-report.md not found"

`/multi-model-review --apply-review` looks under `.cross-review/packages/`. Make sure the reviewer actually wrote its output there — if it wrote to stdout, redirect: `> .cross-review/packages/<pkg>/review-report.md`.

### Report doesn't parse

The reviewer deviated from the schema in [templates/review-report.md](../templates/review-report.md). Options:
- Hand-edit the report to match.
- Re-run the reviewer with a stricter reminder (the template already instructs the exact format).
- Some models (especially chat-tuned) add a preamble — strip it.

### Findings feel too noisy

Raise the confidence floor: `/multi-model-review --apply-review --min-confidence 85`.

### Findings feel too sparse

Lower the floor: `/multi-model-review --apply-review --min-confidence 50`. Or run the package through a *different* reviewer model — different LLMs have very different sensitivities.

### "Template `templates/<reviewer>-review-prompt.md` not found"

Your configured reviewer doesn't have a template. Copy an existing one (see "Adding a new reviewer model" above) or change `reviewer` in config.

## FAQ

**Does this call Codex / Gemini / OpenAI / Google from inside Claude?**  No. The plugin only writes and reads files on disk. You run the other model.

**Can I use it without Spec Kit?**  Yes — ad-hoc mode. Less context for the reviewer, but still useful.

**Can the builder and reviewer be the same model?**  Technically yes, but you lose the cross-model perspective that motivates this plugin. The skill will warn if you configure it this way.

**Can I use more than one reviewer in the same round?**  Yes, manually — see "Multi-reviewer triangulation" above. A merged-report feature is on the roadmap.

**Will this overwrite my code?**  Only through `/multi-model-review --apply-review`, and only after you confirm each edit (critical-severity findings always require explicit confirm).

**Where do I report bugs?**  [github.com/formin/multi-model-review/issues](https://github.com/formin/multi-model-review/issues).
