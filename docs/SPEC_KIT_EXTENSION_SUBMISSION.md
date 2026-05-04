# Spec Kit Extension Submission

This file collects the upstream Spec Kit catalog changes needed after a `v0.1.0` GitHub release is published for this repository.

## Release Prerequisite

Create a release tag that matches `extension.yml`:

```bash
git tag v0.1.0
git push origin v0.1.0
```

The release archive URL used by Spec Kit will be:

```text
https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.0.zip
```

## Community Catalog Entry

Add this entry to `extensions/catalog.community.json` in the `github/spec-kit` repository, keeping the surrounding JSON sorted by extension ID:

```json
"multi-model-review": {
  "name": "Multi-Model Review",
  "id": "multi-model-review",
  "description": "Cross-model Spec Kit handoffs for spec authoring, implementation routing, and review.",
  "author": "formin",
  "version": "0.1.0",
  "download_url": "https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.0.zip",
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
  "updated_at": "2026-05-04T00:00:00Z"
}
```

Also update the top-level `updated_at` field in `extensions/catalog.community.json` when submitting the PR.

## Root README Table Row

Add this row to the root `README.md` Community Extensions table in alphabetical order:

```markdown
| Multi-Model Review | Cross-model Spec Kit handoffs for spec authoring, implementation routing, and review. | `process` | Read+Write | [multi-model-review](https://github.com/formin/multi-model-review) |
```

## PR Checklist

- [ ] `extension.yml` version matches the GitHub release tag.
- [ ] Release archive URL is reachable.
- [ ] README includes Spec Kit extension installation and usage instructions.
- [ ] LICENSE and CHANGELOG are present.
- [ ] Local dev install succeeds with `specify extension add --dev <repo-path>`.
- [ ] Release install succeeds with `specify extension add multi-model-review --from https://github.com/formin/multi-model-review/archive/refs/tags/v0.1.0.zip`.
