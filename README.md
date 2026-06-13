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
/plugin install orchestrate@labset-marketplace
/plugin install connect-go-backend@labset-marketplace
/plugin install connect-ts-backend@labset-marketplace
```

### Updating

Refresh the marketplace to pick up new plugins or updates to existing ones:

```
/plugin marketplace refresh labset-marketplace
```

Update an installed plugin to the latest version:

```
/plugin update spec-delivery@labset-marketplace
/plugin update orchestrate@labset-marketplace
/plugin update connect-go-backend@labset-marketplace
/plugin update connect-ts-backend@labset-marketplace
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

### orchestrate

Agent orchestration layer for spec-delivery. Adds machine-readable status tracking, structured signaling, and autonomous pipeline execution for use with agent orchestrators.

| Skill | Description |
|-------|-------------|
| `/ingest` | Ingest requirements from an external source into a spec, bypassing interactive `/discuss` |
| `/status` | Report machine-readable JSON status of one or all specs |
| `/pipeline` | Run the full lifecycle (ingest -> ship -> audit) autonomously with JSON signals at each stage |
| `/signal` | Write structured state signals (blocked, unblocked, error) to a spec's status.json |

Pipeline states: `ready` -> `shipping` -> `auditing` -> `done` | `audit_failed` | `blocked` | `error`

### connect-go-backend

Incrementally scaffold Connect-RPC backends from proto definitions. Each skill builds on the previous layer, producing a consistent codebase structure rooted at `internal/<provider>/<domain>/<version>/`.

| Skill | Description |
|-------|-------------|
| `/schema` | Generate PostgreSQL schema, sqlc queries, and Atlas migration config from proto messages |
| `/service` | Generate Connect-RPC service `.proto` files with RPCs and request/response types |
| `/handlers` | Generate Connect-RPC handler implementations backed by sqlc stores |
| `/outbox` | Add transactional outbox pattern with River job queue for reliable event delivery |
| `/streaming` | Generate Kafka workers (River to Kafka) and typed consumer group stubs |
| `/mcp` | Generate MCP tool wrappers exposing service operations for Claude integration |
| `/review` | Review structural consistency, naming conventions, and cross-layer alignment |

Skills are designed to work incrementally: `/schema` -> `/service` -> `/handlers` -> `/outbox` -> `/streaming` -> `/mcp`. Run `/review` at any point to verify everything holds together. Each skill assesses the existing codebase before generating, so it can adapt to established patterns or suggest incremental refactors toward the target conventions.

### connect-ts-backend

TypeScript equivalent of connect-go-backend. Incrementally scaffold TypeScript Connect-RPC backends from proto definitions, producing a consistent codebase structure rooted at `src/<provider>/<domain>/<version>/`.

| Skill | Description |
|-------|-------------|
| `/schema` | Generate Drizzle ORM schema, typed queries, and migration config from proto messages |
| `/service` | Generate Connect-RPC service `.proto` files with RPCs and request/response types |
| `/handlers` | Generate Connect-RPC handler implementations backed by Drizzle queries |
| `/outbox` | Add transactional outbox pattern with BullMQ for reliable event delivery |
| `/streaming` | Generate Kafka workers (BullMQ to Kafka) and typed consumer group stubs |
| `/mcp` | Generate MCP tool wrappers exposing service operations for Claude integration |
| `/review` | Review structural consistency, naming conventions, and cross-layer alignment |

Same incremental workflow as connect-go-backend. Stack: Drizzle ORM, Connect-ES, Protobuf-ES, BullMQ, KafkaJS, `@modelcontextprotocol/sdk`.
