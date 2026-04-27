# Code Review Request for Codex via MCP

You are acting as an independent code reviewer. The code below was written by a different agent against the specification in section 2. Your job is to find real problems, not to rewrite the code.

This file is self-contained. Do not rely on prior conversation state.

> Invocation mode: `codex-mcp`
>
> The MCP path is time-boxed. Expect a `micro` package. Focus on the highest-signal issues. If the package is too small to support a trustworthy review, say `needs-full-package`.

## 1. Package profile

- profile: `{{PACKAGE_PROFILE}}`
- scope: `{{PACKAGE_SCOPE}}`
- spec author model: `{{SPEC_AUTHOR_MODEL}}`
- spec author options: `{{SPEC_AUTHOR_OPTIONS}}`
- spec author profile: `{{SPEC_AUTHOR_PROFILE}}`
- implementation model: `{{IMPLEMENTATION_MODEL}}`
- implementation options: `{{IMPLEMENTATION_OPTIONS}}`
- review model: `{{REVIEW_MODEL}}`
- review options: `{{REVIEW_OPTIONS}}`
- review profile: `{{REVIEW_PROFILE}}`
- changed files: `{{CHANGED_FILE_COUNT}}`
- changed lines: `{{CHANGED_LINE_COUNT}}`

## 2. Spec brief

```markdown
{{SPEC_BRIEF}}
```

## 3. Implementation brief

```markdown
{{PLAN_BRIEF}}
```

## 4. Task brief

```markdown
{{TASKS_BRIEF}}
```

## 5. Relevant project rules

```markdown
{{RULES_BRIEF}}
```

## 6. Commit trail

Base: `{{BASE_REF}}`  Head: `{{HEAD_REF}}`

```text
{{LOG_BRIEF}}
```

## 7. Diff manifest

```text
{{DIFF_MANIFEST}}
```

## 8. Focused diff excerpts

```diff
{{DIFF_EXCERPTS}}
```

## 9. Context notes

{{PACKAGE_NOTES}}

## Your task

Produce a review report that follows this schema exactly:

```markdown
# Review report

## Context sufficiency
<one of: sufficient | limited-but-actionable | needs-full-package>

## Verdict
<one of: approve | approve-with-nits | changes-requested | reject>

## Summary
<2-3 sentences, neutral tone>

## Findings

### F1
- severity: <critical | major | minor | info>
- confidence: <0-100>
- location: <path/to/file.ext:LINE or path/to/file.ext>
- summary: <one line>
- detail: <1-3 sentences of evidence>
- suggested_fix: <concrete suggestion or n/a>
```

## Time-boxed checklist

1. Spec alignment
2. Rules adherence
3. Highest-risk correctness issues
4. Security issues
5. Context sufficiency

## What not to flag

- formatter-only issues
- low-value nits
- speculative issues you cannot support inside the time budget

## Confidence scoring

- 100: certain
- 80: high confidence
- 60: likely but not fully verified
- 40: plausible concern
- 20: hunch
