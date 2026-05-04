# claude-marketplace

Claude Code plugin marketplace for the labset org.

## Plugins

### spec-delivery

A spec-driven delivery workflow with four skills:

| Skill | Description |
|-------|-------------|
| `/discuss` | Requirements discussion and milestone planning |
| `/ship` | Build and deliver milestones with production readiness checks |
| `/audit` | Audit spec vs implementation for gaps and divergences |
| `/review` | Review deferred items and recommend future work |

## Setup

Register the marketplace (once per user):

```
/plugin marketplace add labset/claude-marketplace
```

Install the spec-delivery plugin:

```
/plugin install spec-delivery@labset-marketplace
```

### Auto-prompt for a repo

Add to any repo's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "labset-marketplace": {
      "source": {
        "source": "github",
        "repo": "labset/claude-marketplace"
      }
    }
  }
}
```
