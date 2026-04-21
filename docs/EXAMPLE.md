# Worked example

End-to-end example: Claude builds a feature, Codex reviews it, Claude ingests the findings.

## Scenario

You are on `feat/magic-link-auth`.

- builder: Claude Code
- reviewer: Codex CLI
- base ref: `main`
- package profile: `compact`
- spec dir: `specs/042-magic-link-auth`

## Timeline

| Step | Where | Action | Artifact |
|------|-------|--------|----------|
| 1 | Claude | `/multi-model-review:cross-review init` | `.cross-review/config.json` |
| 2 | Claude | implement and commit | source changes |
| 3 | Claude | `/multi-model-review:review-package` | `review-package.md`, `metadata.json` |
| 4 | terminal | `codex exec --file ... > review-report.md` | `review-report.md` |
| 5 | Claude | `/multi-model-review:apply-review` | fixes + `review-state.json` |
| 6 | Claude or shell | commit fixes | updated branch |

## 1. Initialize

```text
/multi-model-review:cross-review init
```

Answers:

- builder: `claude-code`
- reviewer: `codex-cli`
- base ref: `main`
- package profile: `compact`
- spec dir: `specs/042-magic-link-auth`

## 2. Export the package

```text
/multi-model-review:review-package
```

The package is compact-first, so the reviewer sees:

- a spec brief
- a plan brief
- a task brief
- relevant rules
- a grouped diff manifest
- focused diff excerpts
- notes about omissions

Example output:

```text
Package written to:
  .cross-review/packages/20260421-1430-magic-link-auth/review-package.md

Profile:
  compact

Run the reviewer:
  codex exec --file .cross-review/packages/20260421-1430-magic-link-auth/review-package.md > .cross-review/packages/20260421-1430-magic-link-auth/review-report.md
```

## 3. Codex reviews it

Codex writes:

```markdown
## Context sufficiency
limited-but-actionable

## Verdict
changes-requested

## Summary
The magic-link flow is mostly implemented, but the verification path appears to use direct token comparison and the route-level rate limit required by the spec is missing.

## Findings

### F1
- severity: critical
- confidence: 92
- location: src/auth/verify.ts:34
- summary: Token comparison is not constant-time
- detail: The verification path compares secrets using direct equality, which leaks timing information.
- suggested_fix: Use a constant-time comparison helper.

### F2
- severity: major
- confidence: 85
- location: src/auth/routes.ts:12
- summary: Magic-link endpoint has no rate limit
- detail: The spec requires request throttling for the magic-link route.
- suggested_fix: Add route-level rate limiting middleware.
```

Because the context was `limited-but-actionable`, Claude can proceed without re-packaging.

## 4. Claude ingests the report

```text
/multi-model-review:apply-review
```

Claude presents a checklist, reads the cited files, proposes targeted edits, and applies them one at a time after confirmation.

## 5. What happens if compact was not enough?

If Codex had written:

```markdown
## Context sufficiency
needs-full-package
```

Then the next step would be:

```text
/multi-model-review:review-package --full
```

Or, if only one area needed more depth:

```text
/multi-model-review:review-package --paths src/auth
```

That is the core compact-first loop:

1. small package first
2. explicit sufficiency signal
3. fuller package only when needed

## Also see

- [USAGE.md](USAGE.md)
- [TOKEN_EFFICIENCY.md](TOKEN_EFFICIENCY.md)
