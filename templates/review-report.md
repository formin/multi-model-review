# Review report

> Schema for the reviewer agent's output. `/multi-model-review:apply-review` parses this file.

## Context sufficiency

One of: `sufficient` | `limited-but-actionable` | `needs-full-package`

Use `needs-full-package` when the compact package omitted or truncated context that blocks a trustworthy review.

## Verdict

One of: `approve` | `approve-with-nits` | `changes-requested` | `reject`

## Summary

2-3 sentences. Neutral tone. What the change does and whether it appears to satisfy the spec.

## Findings

Zero or more findings, numbered `F1`, `F2`, ...

### F1

- **severity**: `critical` | `major` | `minor` | `info`
- **confidence**: integer `0-100`
- **location**: `path/to/file.ext:LINE` (preferred) or `path/to/file.ext`
- **summary**: one line
- **detail**: 1-3 sentences with evidence from the diff, spec, plan, tasks, or relevant rules
- **suggested_fix**: concrete suggestion, or `n/a`

### F2

...

## Severity definitions

- **critical**: correctness or security bug likely to hit production, or a direct spec violation
- **major**: likely bug, meaningful spec divergence, or relevant rule violation
- **minor**: real but lower-impact issue
- **info**: observation; no action required

## Confidence definitions

- **100**: certain
- **80**: high confidence
- **60**: likely but not fully verified from the package
- **40**: plausible concern
- **20**: hunch

`/multi-model-review:apply-review` drops findings below confidence 70 by default.
