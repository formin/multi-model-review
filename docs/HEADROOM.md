# Headroom-aware review packaging

`multi-model-review` can use [Headroom](https://github.com/chopratejas/headroom) as an optional context-compression layer when Headroom MCP tools are available in the current host.

Headroom compresses tool outputs, logs, files, RAG chunks, and other context before it reaches an LLM. The useful pieces for this plugin are content-aware routing, structure preservation, CCR retrieval hashes, and compression stats.

## Where it fits

The review package stays compact-first:

1. build the spec brief, plan brief, task brief, rules brief, diff manifest, and focused excerpts
2. remove generated noise, lockfile churn, repeated rule text, and low-signal hunks
3. use Headroom only for large raw blocks that remain after manual reduction
4. record retrieval hints and savings in `PACKAGE_NOTES` and `metadata.json`

Headroom is not required. If it is unavailable, the command falls back to the existing manual summaries.

## Modes

| Mode | Behavior |
|------|----------|
| `auto` | Use Headroom MCP compression when callable; otherwise continue manually |
| `off` | Skip Headroom entirely |
| `required` | Abort if Headroom MCP compression is not callable |

Example:

```text
/multi-model-review:review-package --headroom auto
/multi-model-review:review-package --headroom off
/multi-model-review:review-package --headroom required
```

Recommended config:

```json
{
  "context_compression": {
    "provider": "headroom",
    "mode": "auto",
    "min_tokens": 2000,
    "use_mcp": true,
    "require_retrieval_access": false
  }
}
```

## What to preserve

Before compressing any block, keep review-critical anchors visible:

- file paths and line numbers
- public API names and type signatures
- function and class names
- error messages, warnings, IDs, hashes, and migration names
- acceptance criteria, non-goals, and security-sensitive rules
- task IDs and route hints such as `[route:worker]`

## Metadata

When Headroom compresses a block, record:

- source label, such as `git-diff`, `rules`, `tasks`, `log`, or `source-excerpt`
- original and compressed token estimates when available
- savings percentage when available
- transforms when available
- CCR retrieval hash when available
- query hints for `headroom_retrieve`, such as path, symbol, task ID, or error text

Example metadata shape:

```json
{
  "context_compression": {
    "provider": "headroom",
    "mode": "auto",
    "available": true,
    "compressed_blocks": [
      {
        "source": "git-diff",
        "original_tokens": 18000,
        "compressed_tokens": 5200,
        "savings_percent": 71,
        "hash": "abc123",
        "query_hints": ["src/auth", "AuthToken", "migration"]
      }
    ]
  }
}
```

## Reviewer guidance

If the reviewer has access to the same local Headroom store, they may use `headroom_retrieve` with the recorded hash and query hints. If they do not, the package must still contain enough manifest and focused excerpts for a trustworthy review.

When a compressed block hides necessary evidence and retrieval is unavailable, the reviewer should return:

```markdown
## Context sufficiency
needs-full-package
```

Then rerun:

```text
/multi-model-review:review-package --full
```

or narrow the scope:

```text
/multi-model-review:review-package --paths src/auth,src/api
```

## Constraints

- Do not include secrets from `.env`, `.envrc`, credentials files, or similar sources.
- Do not start `headroom proxy`, `headroom wrap`, or another long-running process from the package command.
- Do not assume a direct `headroom compress` CLI subcommand exists; prefer callable Headroom MCP tools.
- Do not use a transient CCR hash as the only copy of review-critical evidence for an external reviewer.
