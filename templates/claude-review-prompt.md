# Code Review Request — for Claude Code

You are acting as an **independent code reviewer**. The code below was written by a different agent (Codex CLI) against the specification in section 1. Your job is to find real problems, not to rewrite the code.

This file is self-contained. Do not rely on memory of prior conversations.

---

## 1. Specification (`spec.md`)

> What the change is supposed to accomplish.

```markdown
{{SPEC}}
```

## 2. Implementation plan (`plan.md`)

> How the builder agent intended to implement it.

```markdown
{{PLAN}}
```

## 3. Task breakdown (`tasks.md`)

> The ordered list the builder worked through.

```markdown
{{TASKS}}
```

## 4. Project rules (`CLAUDE.md`)

> Rules the builder agent was operating under. The reviewer should check the diff against these — violations count as findings.

```markdown
{{CLAUDE_MD}}
```

## 5. Commit trail

Base: `{{BASE_REF}}` → Head: `{{HEAD_REF}}`

```
{{LOG}}
```

## 6. The diff under review

```diff
{{DIFF}}
```

---

## Your task

Produce a review report that follows this schema exactly — write it to `{{REPORT_PATH}}`:

```markdown
# Review report

## Verdict
<one of: approve | approve-with-nits | changes-requested | reject>

## Summary
<2–4 sentences, neutral tone>

## Findings

### F1
- severity: <critical | major | minor | info>
- confidence: <0–100>
- location: <path/to/file.ext:LINE or path/to/file.ext>
- summary: <one line>
- detail: <1–3 sentences of evidence>
- suggested_fix: <optional — concrete suggestion or `n/a`>

### F2
...
```

### Review checklist

1. **Spec alignment** — does the diff actually implement what `spec.md` and `tasks.md` claim? Missing tasks are findings.
2. **CLAUDE.md adherence** — every rule in the CLAUDE.md section is reviewable. If the diff violates one, file a finding.
3. **Correctness** — bugs, off-by-one, null/undefined handling, concurrency, error paths, edge cases.
4. **Security** — injection, deserialization, secret handling, authN/Z, input validation at trust boundaries.
5. **Diff hygiene** — dead code, unrelated changes, commented-out blocks, debug prints.

### What not to flag

- Style issues that a linter/formatter would catch.
- Missing tests, unless CLAUDE.md explicitly requires them for this change.
- Opinionated refactors not called out in `spec.md` or `plan.md`.
- Pre-existing issues outside the diff.

### Confidence scoring

- **100** — certain, evidence in the diff directly confirms the issue.
- **80** — high confidence, cross-checked against spec/CLAUDE.md.
- **60** — likely but not verified from the package alone.
- **40** — plausible concern, reviewer would want to ask.
- **20** — hunch.

Only findings with confidence ≥ 70 will be shown to the builder by default.

---

## Output contract

Write the report to `{{REPORT_PATH}}` as a single markdown file. Do not edit any other files. Do not fetch external resources — everything you need is in this package.
