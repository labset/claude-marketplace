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
| `/config` | Generate a centralised configuration package under `internal/config/` with required/optional env var resolution |
| `/db-schema` | Generate PostgreSQL schema, sqlc queries, and Atlas migration config from proto messages |
| `/protos` | Generate Connect-RPC proto definitions (entity models, service RPCs, request/response types) |
| `/api-handlers` | Generate Connect-RPC handler implementations backed by sqlc stores |
| `/server` | Generate a server entrypoint under `cmd/` — supports `api` (HTTP) and `mcp` (stdio) transports |
| `/outbox` | Add transactional outbox pattern with River job queue for reliable event delivery |
| `/streaming` | Generate Kafka workers (River to Kafka) and typed consumer group stubs |
| `/mcp-tools` | Generate MCP tool wrappers exposing service operations for Claude integration |
| `/verify` | Verify structural consistency, naming conventions, and cross-layer alignment |

Skills are designed to work incrementally: `/config` -> `/db-schema` -> `/protos` -> `/api-handlers` -> `/server api` -> `/outbox` -> `/streaming` -> `/mcp-tools` -> `/server mcp`. Run `/verify` at any point to verify everything holds together.

