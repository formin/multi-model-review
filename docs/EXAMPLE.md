# Worked example

A full end-to-end walkthrough: Claude builds a feature, Codex reviews it, Claude ingests the findings.

## Scenario

You are on a feature branch `feat/magic-link-auth` that implements magic-link login for a web app. You scaffolded and implemented the change inside Claude Code. Before opening a pull request you want a *different* model to review it — you have Codex CLI installed, so you will use Codex as the reviewer.

The repo has Spec Kit initialized, so the feature spec lives at `specs/042-magic-link-auth/` (`spec.md`, `plan.md`, `tasks.md`).

## 1. Initialize config

Inside Claude Code:

```
/multi-model-review:cross-review init
```

Answer the prompts:

| Prompt             | Answer                         |
|--------------------|--------------------------------|
| builder agent      | `claude-code`                  |
| reviewer agent     | `codex-cli`                    |
| base ref           | `main`                         |
| spec-kit feature   | `specs/042-magic-link-auth`    |

The plugin writes `.cross-review/config.json`:

```json
{
  "builder": "claude-code",
  "reviewer": "codex-cli",
  "base_ref": "main",
  "spec_dir": "specs/042-magic-link-auth"
}
```

and appends `.cross-review/` to `.gitignore` (unless you opt in to committing it).

## 2. Check status at any time

```
/multi-model-review:cross-review
```

Expected output:

```
Builder:  claude-code
Reviewer: codex-cli
Base ref: main
Spec:     specs/042-magic-link-auth

Packages:
  (none yet)
```

## 3. Build your change

Implement the feature as you normally would — Spec Kit path (`/speckit.implement`) or ad-hoc. Commit on the feature branch; uncommitted work goes into the diff too but will not appear in `git log`.

## 4. Export the review package

```
/multi-model-review:review-package
```

The plugin gathers `spec.md`, `plan.md`, `tasks.md`, `git diff main...HEAD`, `git log main...HEAD --oneline`, and every applicable `CLAUDE.md`, then wraps them in the Codex-specific reviewer template. Example output:

```
Package written to:
  .cross-review/packages/20260417-1445-magic-link-auth/review-package.md

Run the reviewer (separate terminal):
  codex exec --file .cross-review/packages/20260417-1445-magic-link-auth/review-package.md \
    > .cross-review/packages/20260417-1445-magic-link-auth/review-report.md

When done, run /multi-model-review:apply-review.
```

## 5. Run Codex yourself

Copy and paste the printed command into a separate terminal. The plugin **does not** invoke Codex for you — this keeps auth, cost, and rate limits in your hands.

Codex reads the self-contained package, writes its findings to `review-report.md` following the schema in [templates/review-report.md](../templates/review-report.md). A trimmed sample:

```markdown
## Verdict
changes-requested

## Summary
The magic-link flow is implemented but the token-verification step
uses non-constant-time comparison, and spec §4.2 mandates rate
limiting that is not present.

## Findings
### F1
- severity: critical
- confidence: 92
- location: src/auth/verify.ts:34
- summary: Token comparison uses non-constant-time equality
- detail: `token === storedToken` leaks timing information. An
  attacker can infer the token byte-by-byte from response latency.
- suggested_fix: Replace with `crypto.timingSafeEqual(Buffer.from(token), Buffer.from(storedToken))`.

### F2
- severity: major
- confidence: 85
- location: src/auth/routes.ts:12
- summary: Magic-link endpoint has no rate limit
- detail: Spec §4.2 requires 5 req/min per IP on /auth/magic. The
  route is exposed without any limiter middleware.
- suggested_fix: Mount `rateLimit({ windowMs: 60_000, max: 5 })` on the route.
```

## 6. Ingest the review

Back in Claude Code:

```
/multi-model-review:apply-review
```

Claude parses the report, drops findings with `confidence < 70` by default, and presents a numbered checklist:

```
2 findings above confidence 70 (1 critical, 1 major)

[1] critical (conf 92) — src/auth/verify.ts:34
    Token comparison uses non-constant-time equality

[2] major (conf 85) — src/auth/routes.ts:12
    Magic-link endpoint has no rate limit

Apply which? (all | 1,2 | 1 | skip)
```

Pick `all` or specific numbers. For each accepted finding, Claude reads the cited file, proposes an `Edit` that implements `suggested_fix`, and waits for your confirmation before applying.

`severity: critical` findings always require an explicit confirm per finding, even under `all`.

State is written to `.cross-review/packages/20260417-1445-magic-link-auth/review-state.json` so re-runs know which findings were already applied.

## 7. Iterate

After applying fixes, commit, then run `/multi-model-review:review-package` again for a second round. Each package is timestamped so you keep an audit trail of review rounds. Consider rotating reviewer between rounds — e.g. round 1 = Codex, round 2 = Gemini — to surface different classes of issues.

## Common variations

### Scope to a specific spec

When multiple feature directories live under `specs/`:

```
/multi-model-review:review-package 042-magic-link-auth
```

### Diff against a different base

```
/multi-model-review:review-package --base release/2026-q2
```

### Raise or lower the confidence floor during apply

```
/multi-model-review:apply-review --min-confidence 85    # only high-confidence findings
/multi-model-review:apply-review --min-confidence 0     # see every finding including nits
```

### Ingest a specific package directory

```
/multi-model-review:apply-review .cross-review/packages/20260417-1445-magic-link-auth
```

### Triangulate with two reviewers

```bash
PKG=.cross-review/packages/20260417-1445-magic-link-auth

codex  exec --file $PKG/review-package.md > $PKG/review-report-codex.md
gemini --file $PKG/review-package.md       > $PKG/review-report-gemini.md
```

Rename one to `review-report.md` and run `/multi-model-review:apply-review $PKG` to walk through that one; merge the other's findings by hand. Findings present in both reports are almost certainly real — findings unique to one reviewer are where cross-model value lives, but treat them with healthy skepticism.

### Swap the builder/reviewer roles

Edit `.cross-review/config.json` — swap `builder` and `reviewer` — or re-run `/multi-model-review:cross-review init`. The next `/multi-model-review:review-package` picks the template for the new reviewer automatically.

## Also see

- [USAGE.md](USAGE.md) — step-by-step reference, troubleshooting, FAQ
- [../README.md](../README.md) — install, commands, design notes
- [../templates/review-report.md](../templates/review-report.md) — schema every reviewer must follow
