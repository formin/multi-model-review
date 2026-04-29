# Development Spec Authoring Request

You are the spec authoring model for this feature.

This file is self-contained. Do not rely on prior conversation state.

## 1. Model routing

- spec author model: `{{SPEC_AUTHOR_MODEL}}`
- spec author options: `{{SPEC_AUTHOR_OPTIONS}}`
- spec author profile: `{{SPEC_AUTHOR_PROFILE}}`
- implementation model: `{{IMPLEMENTATION_MODEL}}`
- implementation options: `{{IMPLEMENTATION_OPTIONS}}`
- subagent routing: `{{SUBAGENT_ROUTING}}`

Write the development artifacts so the implementation model can execute them efficiently. Respect the implementation options when sizing tasks. When the implementation model is `claude-sonnet-4.6`, prefer clear task slices, explicit acceptance checks, and minimal ambiguity so implementation can stay token-conscious.

If subagent routing is enabled, write `tasks.md` so the builder can delegate safely:

- `route:scout` for read-only discovery, dependency mapping, and context summaries
- `route:heavy-planner` for cross-cutting design, migration, security-sensitive, or ambiguous work before edits
- `route:worker` for scoped implementation and test edits
- `route:review-checker` for read-only local preflight after an implementation slice

Use only route hints. Do not invent model IDs outside the routing contract.

## 2. Feature

- slug: `{{FEATURE_SLUG}}`

```markdown
{{FEATURE_BRIEF}}
```

## 3. Project context

```markdown
{{PROJECT_CONTEXT}}
```

## 4. Existing Spec Kit artifacts

### Existing spec.md

```markdown
{{EXISTING_SPEC}}
```

### Existing plan.md

```markdown
{{EXISTING_PLAN}}
```

### Existing tasks.md

```markdown
{{EXISTING_TASKS}}
```

## 5. Output notes

{{OUTPUT_NOTES}}

## Your task

Produce or update the Spec Kit artifacts for implementation. Do not implement production code.

Return the output as file blocks with exact target paths:

```markdown
## specs/<slug>/spec.md
<complete spec content>

## specs/<slug>/plan.md
<complete plan content>

## specs/<slug>/tasks.md
<complete task list>
```

## Requirements

- Preserve explicit user requirements and constraints.
- Make acceptance criteria testable.
- Identify non-goals and edge cases.
- Include implementation risks and integration points in `plan.md`.
- Break `tasks.md` into small, ordered tasks that `{{IMPLEMENTATION_MODEL}}` can perform in focused passes.
- When subagent routing is enabled, include a route hint on each implementation task, for example `[route:worker]`, `[route:scout]`, `[route:heavy-planner]`, or `[route:review-checker]`.
- Mark any unresolved product or technical questions clearly.
- Avoid broad rewrites outside the requested feature.
