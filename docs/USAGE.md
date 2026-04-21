# Usage guide

Step-by-step walkthrough for `multi-model-review`.

## 1. Prerequisites

- Claude Code installed and authenticated
- `git` on `PATH`
- at least one reviewer model installed locally
- Spec Kit is recommended, but not required

Supported reviewer examples:

- Codex CLI
- Codex MCP
- Gemini CLI
- another Claude session

## 2. Initialize the workflow

Inside the target repo:

```text
/multi-model-review:cross-review init
```

Typical answers:

| Prompt | Example |
|--------|---------|
| builder | `claude-code` |
| reviewer | `codex-auto` |
| base ref | `main` |
| package profile | `compact` |
| spec dir | `specs/001-auth-rework` |

Example config:

```json
{
  "builder": "claude-code",
  "reviewer": "codex-auto",
  "base_ref": "main",
  "spec_dir": "specs/001-auth-rework",
  "package_profile": "compact"
}
```

## 3. Build the feature

Implement the feature however you normally work:

- Spec Kit path: `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`
- ad-hoc path: edit code directly

Commit the branch when possible. The reviewer can inspect uncommitted changes in the diff, but the commit trail is more useful when commits exist.

## 4. Export the review package

```text
/multi-model-review:review-package
```

Optional variants:

```text
/multi-model-review:review-package --full
/multi-model-review:review-package --paths src/auth,src/api
/multi-model-review:review-package 001-auth-rework --base release/2026-q2
```

What the command now does:

1. resolves the feature slug and base ref
2. profiles the diff
3. builds a compact package by default
4. writes:

```text
.cross-review/packages/<YYYYMMDD-HHMM>-<slug>/
  review-package.md
  metadata.json
```

The compact package contains summaries, a diff manifest, focused excerpts, and omission notes. It does not dump every raw artifact by default.

## 5. Run the reviewer yourself

Examples:

```bash
PKG=.cross-review/packages/20260421-1400-auth-rework

codex exec --file $PKG/review-package.md > $PKG/review-report.md
gemini --file $PKG/review-package.md > $PKG/review-report.md
claude -p "$(cat $PKG/review-package.md)" > $PKG/review-report.md
```

For `codex-mcp`, the skill may use the MCP path inline for very small packages. Anything longer should move to the CLI path.

## 6. Review report structure

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

## 7. Ingest the report

```text
/multi-model-review:apply-review
```

Optional:

```text
/multi-model-review:apply-review --min-confidence 85
/multi-model-review:apply-review .cross-review/packages/20260421-1400-auth-rework
```

Default behavior:

1. parse the report
2. stop early if the report says `needs-full-package`
3. drop findings with confidence below 70
4. present a checklist
5. apply accepted findings one at a time

## 8. When to rerun with `--full`

Use `--full` when:

- the reviewer explicitly asks for it
- the diff is too cross-cutting for focused excerpts
- rule interpretation depends on large omitted appendices
- the review is security-sensitive and you want maximum raw context

Do not default to `--full` for every review. The compact package is the normal path.

## 9. When to use `--paths`

Use `--paths` when the full branch is too large but the issue is local:

- `src/auth`
- `src/payments`
- `db/migrations`

This often gives better signal than switching blindly to a huge `--full` package.

## 10. RTK-inspired behavior

The package strategy borrows the same ideas that RTK applies to shell output:

- filter noise
- group related items
- truncate repetitive output
- deduplicate repeated structure

See [TOKEN_EFFICIENCY.md](TOKEN_EFFICIENCY.md) for the detailed mapping.

## 11. Troubleshooting

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
