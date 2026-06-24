# claude-marketplace

Claude Code plugin marketplace for the labset org.

## Setup

Register the marketplace (once per user):

```
/plugin marketplace add labset/claude-marketplace
```

Install a plugin:

```
/plugin install spec-delivery@labset-marketplace
/plugin install connect-go-backend@labset-marketplace
```

### Updating

Refresh the marketplace to pick up new plugins or updates to existing ones:

```
/plugin marketplace refresh labset-marketplace
```

Update an installed plugin to the latest version:

```
/plugin update spec-delivery@labset-marketplace
/plugin update connect-go-backend@labset-marketplace
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

## Plugins

### spec-delivery

A spec-driven delivery workflow with four skills:

| Skill | Description |
|-------|-------------|
| `/discuss` | Requirements discussion and milestone planning |
| `/ship` | Build and deliver milestones with production readiness checks |
| `/audit` | Audit spec vs implementation for gaps and divergences |
| `/review` | Review deferred items and recommend future work |

### connect-go-backend

Incrementally scaffold Connect-RPC backends from proto definitions. Each skill builds on the previous layer, producing a consistent codebase structure rooted at `internal/<provider>/<domain>/<version>/`.

| Skill | Description |
|-------|-------------|
| `/db-schema` | Generate PostgreSQL schema, sqlc queries, and Atlas migration config from proto messages |
| `/protos` | Generate Connect-RPC proto definitions (entity models, service RPCs, request/response types) |
| `/api-handlers` | Generate Connect-RPC handler implementations backed by sqlc stores |
| `/outbox` | Add transactional outbox pattern with River job queue for reliable event delivery |
| `/streaming` | Generate Kafka workers (River to Kafka) and typed consumer group stubs |
| `/mcp-tools` | Generate MCP tool wrappers exposing service operations for Claude integration |
| `/verify` | Verify structural consistency, naming conventions, and cross-layer alignment |

Skills are designed to work incrementally: `/db-schema` -> `/protos` -> `/api-handlers` -> `/outbox` -> `/streaming` -> `/mcp-tools`. Run `/verify` at any point to verify everything holds together. Each skill assesses the existing codebase before generating, so it can adapt to established patterns or suggest incremental refactors toward the target conventions.

