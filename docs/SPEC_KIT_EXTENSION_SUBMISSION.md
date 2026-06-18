# Spec Kit Extension Submission Issue Payload

This file tracks the upstream Spec Kit catalog update for `multi-model-review`
`v0.1.2`.

The current Spec Kit publishing guide requires community catalog updates to go
through a new **Extension Submission** issue. Do not open a pull request that
edits `extensions/catalog.community.json` directly.

## Release Prerequisite

Create a release tag that matches `extension.yml`:

```bash
git tag v0.1.2
git push origin v0.1.2
```

The release archive URL used by Spec Kit is:

```text
https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.2.zip
```

## Issue Title

```text
[Extension]: Update multi-model-review to 0.1.2
```

## Template Fields

| Field | Value |
|-------|-------|
| Extension ID | `multi-model-review` |
| Extension Name | `Multi-Model Review` |
| Version | `0.1.2` |
| Description | Cross-model Spec Kit handoffs for spec authoring, implementation routing, and review. |
| Author | `formin` |
| Repository URL | `https://github.com/formin/multi-model-review` |
| Download URL | `https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.2.zip` |
| License | `MIT` |
| Homepage | `https://github.com/formin/multi-model-review` |
| Documentation URL | `https://github.com/formin/multi-model-review/blob/main/README.md` |
| Changelog URL | `https://github.com/formin/multi-model-review/blob/main/CHANGELOG.md` |
| Required Spec Kit Version | `>=0.2.0` |
| Number of Commands | `4` |
| Number of Hooks | `0` |
| Tags | `review, workflow, multi-model, spec-driven-development, code` |

Required tools:

```markdown
- git - required
- codex - optional
- gemini - optional
- claude - optional
```

Key features:

```markdown
- Route Spec Kit spec authoring, implementation metadata, and review across different models.
- Export compact-first cross-model review packages from Spec Kit artifacts and git diffs.
- Support Codex CLI, Codex MCP, Claude, Gemini, Hermes, and custom CLI reviewer templates.
- Document Codex OSS/local provider review and spec-authoring flows through `codex exec --oss`.
- Keep review findings confidence-gated and ingest accepted findings into remediation state.
```

Example usage:

````markdown
```bash
specify extension add multi-model-review --from https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.2.zip
```

```text
/speckit.multi-model-review.cross-review init --spec codex-5.5:xhigh@normal --spec-heavy opus-4.7:1m@max --dev sonnet-4.6@high --review codex-5.5:high@normal --subagents auto
/speckit.multi-model-review.review-package 001-auth-rework
```
````

Testing details to paste after release verification:

```markdown
**Tested on:**
- Windows with Spec Kit CLI available on PATH

**Test scenarios:**
1. `specify extension add --dev <repo-path>`
2. `specify extension add multi-model-review --from https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.2.zip`
3. Verified installed manifest version is `0.1.2`
4. Verified command files are included in the installed extension
```

## Proposed Catalog Entry

```json
{
  "multi-model-review": {
    "name": "Multi-Model Review",
    "id": "multi-model-review",
    "description": "Cross-model Spec Kit handoffs for spec authoring, implementation routing, and review.",
    "author": "formin",
    "version": "0.1.2",
    "download_url": "https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.2.zip",
    "repository": "https://github.com/formin/multi-model-review",
    "homepage": "https://github.com/formin/multi-model-review",
    "documentation": "https://github.com/formin/multi-model-review/blob/main/README.md",
    "changelog": "https://github.com/formin/multi-model-review/blob/main/CHANGELOG.md",
    "license": "MIT",
    "requires": {
      "speckit_version": ">=0.2.0",
      "tools": [
        {
          "name": "git",
          "required": true
        },
        {
          "name": "codex",
          "required": false
        },
        {
          "name": "gemini",
          "required": false
        },
        {
          "name": "claude",
          "required": false
        }
      ]
    },
    "provides": {
      "commands": 4,
      "hooks": 0
    },
    "tags": [
      "review",
      "workflow",
      "multi-model",
      "spec-driven-development",
      "code"
    ],
    "verified": false,
    "downloads": 0,
    "stars": 0,
    "created_at": "2026-05-04T00:00:00Z",
    "updated_at": "2026-06-19T00:00:00Z"
  }
}
```

## Submission Checklist

- [ ] `extension.yml` version matches the GitHub release tag.
- [ ] Release archive URL is reachable.
- [ ] README includes Spec Kit extension installation and usage instructions.
- [ ] LICENSE and CHANGELOG are present.
- [ ] Local dev install succeeds with `specify extension add --dev <repo-path>`.
- [ ] Release install succeeds with `specify extension add multi-model-review --from https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.2.zip`.
- [ ] Extension Submission issue mentions that this updates the existing catalog entry.
