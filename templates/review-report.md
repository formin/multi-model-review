# Review report

> Schema for the reviewer agent's output. `/apply-review` parses this file.
> The reviewer must write its report to `.cross-review/packages/<pkg>/review-report.md`
> following this exact structure.

## Verdict

One of: `approve` | `approve-with-nits` | `changes-requested` | `reject`

## Summary

2–4 sentences. Neutral tone. What the change does and whether it achieves the spec.

## Findings

Zero or more findings, numbered `F1`, `F2`, ...

### F1

- **severity**: `critical` | `major` | `minor` | `info`
- **confidence**: integer 0–100
- **location**: `path/to/file.ext:LINE` (preferred) or `path/to/file.ext`
- **summary**: one line
- **detail**: 1–3 sentences with evidence from the diff, spec, or CLAUDE.md
- **suggested_fix**: concrete suggestion, or `n/a`

### F2

...

## Severity definitions

- **critical** — correctness/security bug that will hit in production, or a direct spec violation.
- **major** — likely bug, meaningful spec divergence, or a CLAUDE.md rule violation.
- **minor** — real but low-impact issue (edge case, maintainability).
- **info** — observation; no action required.

## Confidence definitions

- **100** — certain, evidence in the diff directly confirms.
- **80** — high confidence, cross-checked against spec/CLAUDE.md.
- **60** — likely but not verified from the package alone.
- **40** — plausible concern, reviewer would want to ask.
- **20** — hunch.

`/apply-review` drops findings below confidence 70 by default.
