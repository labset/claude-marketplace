---
description: Generate PostgreSQL schema, sqlc queries, and Atlas migration config from proto message definitions. Use when the user wants to create or update the database layer for their proto entities.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /db-schema - Database Schema and Query Generation

You are generating the database layer for a Connect-RPC backend. Your goal is to read proto message definitions and produce PostgreSQL schema, sqlc query definitions, sqlc configuration, and Atlas migration configuration.

Before generating, scan for existing code that overlaps with what this skill produces. If existing patterns are found, present divergences and ask the user to confirm a plan before proceeding. If no existing code is found, proceed directly. After generating, verify expected files exist with correct package declarations and imports, then present a summary. If target files already exist, ask the user before overwriting.

## Setup

1. Determine the proto source:
   - If the user provides a path (e.g. `/db-schema protos/acme/inventory/v1/`), use it
   - Otherwise, search for `.proto` files and ask the user which messages to target
2. Read the proto files and identify entity messages (messages with `id`, `created_at`, `updated_at` fields, or `ROLE_ENTITY` options, or `labset.data.v1.Entity` embedding)
3. Resolve the package path from the proto `package` declaration:
   - `acme.inventory.v1` becomes `internal/acme/inventory/v1/`
   - All subpackages (`sql/`, `db/`, `api/`, `outbox/`, `workers/`, `consumers/`, `mcp/`) nest under this path
4. Read `go.mod` for the module path

## Schema Generation

Generate `sql/schema.sql` in the output directory.

- Derive schema name from proto package: `acme.inventory.v1` becomes `acme_inventory_v1`
- Begin with `CREATE SCHEMA IF NOT EXISTS <schema_name>;`
- Table names: `<schema_name>.<entity_snake>` (e.g. `acme_inventory_v1.product`)

Each entity table includes:
- **Entity columns**: `id UUID PRIMARY KEY`, `created_at TIMESTAMPTZ NOT NULL`, `updated_at TIMESTAMPTZ NOT NULL`, `deleted_at TIMESTAMPTZ` (nullable, for soft deletes)
- **User-defined fields** mapped from proto types:

| Proto Type | PostgreSQL Type |
|---|---|
| `string` | `TEXT` |
| `bytes` | `BYTEA` |
| `bool` | `BOOLEAN` |
| `int32`, `sint32`, `sfixed32`, `uint32`, `fixed32` | `INTEGER` |
| `int64`, `sint64`, `sfixed64`, `uint64`, `fixed64` | `BIGINT` |
| `float` | `REAL` |
| `double` | `DOUBLE PRECISION` |
| `google.protobuf.Timestamp` | `TIMESTAMPTZ` |
| `google.protobuf.Duration` | `INTERVAL` |
| `google.protobuf.Struct` / `Value` | `JSONB` |
| `enum` | `TEXT` |
| `repeated <scalar>` | Array type (e.g. `TEXT[]`) |
| `repeated <message>`, nested message, `map` | `JSONB` |
| `oneof` variants | Nullable columns, one per variant |
| Reference fields (messages ending in `Ref`) | `UUID` with foreign key |

All columns are `NOT NULL` by default except `deleted_at`, `oneof` variants, and wrapper types. Skip the `entity` field itself (its columns are inlined).

## Query Generation

Generate `sql/queries/<entity_snake>.sql` per entity with sqlc annotations, **named parameters** (`@param`), and **explicit column lists** (not `SELECT *`).

Required queries per entity:
- `Get<Model> :one` — by `id`, filtered by `deleted_at IS NULL`
- `List<Model>s :many` — ordered by `created_at DESC`, filtered by `deleted_at IS NULL`
- `List<Model>sPaginated :many` — cursor-based using zero UUID sentinel: `(@cursor::uuid = '00000000-0000-0000-0000-000000000000'::uuid OR id > @cursor)`, ordered by `id`, with `LIMIT @page_size`
- `Create<Model> :one` — `INSERT ... RETURNING *`
- `Update<Model> :one` — `UPDATE ... SET ... RETURNING *`, filtered by `deleted_at IS NULL`
- `SoftDelete<Model> :execrows` — sets `deleted_at = NOW()`, filtered by `deleted_at IS NULL`
- `Delete<Model> :exec` — hard delete

## sqlc Configuration

Generate `sqlc.yaml`:

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "sql/queries"
    schema: "sql/schema.sql"
    gen:
      go:
        package: "db"
        out: "db"
        sql_package: "pgx/v5"
        overrides:
          - db_type: "uuid"
            go_type:
              import: "github.com/gofrs/uuid/v5"
              type: "UUID"
          - db_type: "uuid"
            nullable: true
            go_type:
              import: "github.com/gofrs/uuid/v5"
              type: "NullUUID"
```

## Atlas Migration Configuration

Generate `atlas.hcl`:

```hcl
env "local" {
  src = [
    "file://sql/baseline.sql",
    "file://sql/schema.sql",
  ]
  dev = "docker://postgres/17-alpine/dev"
  migration {
    dir    = "file://migrations"
    format = goose
  }
}
```

The default migration format is `goose`. If the user specifies a different format (e.g. `golang-migrate`, `flyway`), use that instead.

Generate `sql/baseline.sql`:

```sql
-- This prevents Atlas from attempting to drop the public schema
CREATE SCHEMA IF NOT EXISTS public;
```

## Embedded Migrations

Generate `migrations.go` as a sibling to `atlas.hcl`:

```go
package <version>

import "embed"

//go:embed migrations/*.sql
var Migrations embed.FS
```

This embeds the migration files for use at runtime (e.g. with goose). The `package` declaration must match the directory name (e.g. `package v1`).

## Go Generate File

Generate `generate.go` as a sibling to `atlas.hcl` and `sqlc.yaml`:

```go
package <version>

//go:generate sqlc generate
//go:generate atlas migrate diff --env local
```

This allows running `go generate ./internal/<provider>/<domain>/<version>/` to execute both sqlc code generation and Atlas migration diffing in one command.

## Rules

- Preserve existing sql files that are not entity-related
- Do not generate Go code (handled by sqlc tooling and subsequent skills)
