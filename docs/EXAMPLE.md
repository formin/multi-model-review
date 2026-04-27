# Worked example

End-to-end example: Codex writes the development specs, Claude Sonnet implements the feature, Codex reviews it, and Claude ingests the findings.

## Scenario

You are on `feat/magic-link-auth`.

- builder: Claude Code
- spec default: `codex-5.5:xhigh@normal`
- heavy spec default: `opus-4.7:1m@max`
- implementation default: `sonnet-4.6@high`
- review default: `codex-5.5:high@normal`
- reviewer surface: Codex CLI
- base ref: `main`
- package profile: `compact`
- spec dir: `specs/042-magic-link-auth`

## Timeline

| Step | Where | Action | Artifact |
|------|-------|--------|----------|
| 1 | Claude | `/multi-model-review:cross-review init` | `.cross-review/config.json` |
| 2 | Claude | `/multi-model-review:spec-handoff` | `spec-authoring-prompt.md`, `metadata.json` |
| 3 | terminal | run Codex 5.5 on the spec handoff | `spec-output.md` |
| 4 | Claude Sonnet 4.6 | implement and commit | source changes |
| 5 | Claude | `/multi-model-review:review-package` | `review-package.md`, `metadata.json` |
| 6 | terminal | `codex exec -m codex-5.5 --file ... > review-report.md` | `review-report.md` |
| 7 | Claude | `/multi-model-review:apply-review` | fixes + `review-state.json` |
| 8 | Claude or shell | commit fixes | updated branch |

## 1. Initialize

```text
/multi-model-review:cross-review init \
  --spec codex-5.5:xhigh@normal \
  --spec-heavy opus-4.7:1m@max \
  --dev sonnet-4.6@high \
  --review codex-5.5:high@normal
```

Answers:

- builder: `claude-code`
- spec default: `codex-5.5:xhigh@normal`
- heavy spec default: `opus-4.7:1m@max`
- implementation default: `sonnet-4.6@high`
- review default: `codex-5.5:high@normal`
- reviewer: `codex-cli`
- base ref: `main`
- package profile: `compact`
- spec dir: `specs/042-magic-link-auth`

## 2. Export the spec handoff

```text
/multi-model-review:spec-handoff 042-magic-link-auth --spec-model codex-5.5:xhigh@normal
```

Example output:

```text
Spec handoff written to:
  .cross-review/spec-handoffs/20260421-1410-magic-link-auth/spec-authoring-prompt.md

Spec author model:
  codex-5.5

Spec author options:
  intelligence=very-high, reasoning=xhigh, speed=normal

Implementation model:
  claude-sonnet-4.6

Implementation options:
  workload=high, allow_silent_upgrade=false
```

Run Codex 5.5 against the prompt and review the returned `spec.md`, `plan.md`, and `tasks.md` file blocks before applying them.

## 3. Export the review package

```text
/multi-model-review:review-package --review-model codex-5.5:high@normal
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
  codex exec -m codex-5.5 --file .cross-review/packages/20260421-1430-magic-link-auth/review-package.md > .cross-review/packages/20260421-1430-magic-link-auth/review-report.md
```

## 4. Codex reviews it

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

## 5. Claude ingests the report

```text
/multi-model-review:apply-review
```

Claude presents a checklist, reads the cited files, proposes targeted edits, and applies them one at a time after confirmation.

## 6. What happens if compact was not enough?

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
