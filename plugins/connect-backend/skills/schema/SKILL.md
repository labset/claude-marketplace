---
description: Generate PostgreSQL schema, sqlc queries, and Atlas migration config from proto message definitions. Use when the user wants to create or update the database layer for their proto entities.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /schema - Database Schema and Query Generation

You are generating the database layer for a Connect-RPC backend. Your goal is to read proto message definitions and produce PostgreSQL schema, sqlc query definitions, sqlc configuration, and Atlas migration configuration.

## Setup

1. Determine the proto source:
   - If the user provides a path (e.g. `/schema protos/acme/inventory/v1/`), use it
   - Otherwise, search for `.proto` files in the project and ask the user which messages to target
2. Read the proto files and identify entity messages:
   - An entity message is one that embeds a `clarity.plugin.v1.Entity` field or has fields like `id`, `created_at`, `updated_at` that indicate it is a database-backed entity
   - If no clear entity marker exists, ask the user which messages represent database entities
3. Resolve the package path — this convention is critical and must be followed by all connect-backend skills:
   - Read the proto file's `package` declaration (e.g. `package acme.inventory.v1;`)
   - The package segments map directly to the folder path: `acme.inventory.v1` becomes `internal/acme/inventory/v1/`
   - The output directory for this skill is `internal/<provider>/<domain>/<version>/`
   - All subpackages (`sql/`, `db/`, `api/`, `outbox/`, `workers/`, `consumers/`, `mcp/`) are nested under this path
   - Example: proto package `acme.inventory.v1` produces:
     ```
     internal/acme/inventory/v1/
     ├── sql/
     ├── db/
     ├── api/
     ├── outbox/
     ├── workers/
     ├── consumers/
     └── mcp/
     ```
   - If the project already has generated code from a previous skill run, use the same path
4. Read `go.mod` to determine the Go module path — all Go imports for subpackages use `<module>/internal/<provider>/<domain>/<version>/<subpackage>`

## Phase 1: Schema Generation

Generate `sql/schema.sql` in the output directory.

### Schema naming

- Derive the schema name from the proto package: `acme.inventory.v1` becomes `acme_inventory_v1`
- Begin with `CREATE SCHEMA IF NOT EXISTS <schema_name>;`

### Entity table structure

For each entity message, generate a `CREATE TABLE` statement with:

- **Entity columns** (always present, inlined):
  - `id UUID PRIMARY KEY`
  - `created_at TIMESTAMPTZ NOT NULL`
  - `updated_at TIMESTAMPTZ NOT NULL`
  - `deleted_at TIMESTAMPTZ` (nullable, for soft deletes)
- **User-defined fields** mapped from proto types using these rules:

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
| `enum` | `TEXT` with `CHECK` constraint listing valid enum values |
| `repeated <scalar>` | Array type (e.g. `TEXT[]`, `INTEGER[]`) |
| `repeated <message>`, nested message, `map` | `JSONB` |
| `oneof` variants | Nullable columns, one per variant |
| Reference fields (messages ending in `Ref`) | `UUID` |

- **Foreign keys**: If a field references another entity (field name ending in `_ref` or message type ending in `Ref`), generate a `UUID` column named `<field>_id` with a `REFERENCES <schema>.<table>(id)` constraint
- **NOT NULL**: All columns are `NOT NULL` by default except `deleted_at`, `oneof` variants, and wrapper types (`google.protobuf.*Value`)
- Skip the `entity` field itself from the proto message (its columns are inlined as the entity columns above)

### Table naming

- Table names are snake_case of the message name: `Product` becomes `product`, `OrderItem` becomes `order_item`
- Fully qualified: `<schema_name>.<table_name>`

## Phase 2: Query Generation

Generate `sql/queries/<entity_snake>.sql` for each entity.

Each query file must use sqlc annotations. Generate the following queries:

```sql
-- name: Get<Model> :one
SELECT * FROM <schema>.<table>
WHERE id = $1 AND deleted_at IS NULL;

-- name: List<Model>s :many
SELECT * FROM <schema>.<table>
WHERE deleted_at IS NULL
ORDER BY created_at DESC;

-- name: List<Model>sPaginated :many
SELECT * FROM <schema>.<table>
WHERE deleted_at IS NULL AND id < $1
ORDER BY id DESC
LIMIT $2;

-- name: Create<Model> :one
INSERT INTO <schema>.<table> (<all columns except deleted_at>)
VALUES (<positional params>)
RETURNING *;

-- name: Update<Model> :one
UPDATE <schema>.<table>
SET <all user fields + updated_at>, updated_at = $N
WHERE id = $1 AND deleted_at IS NULL
RETURNING *;

-- name: SoftDelete<Model> :execrows
UPDATE <schema>.<table>
SET deleted_at = NOW()
WHERE id = $1 AND deleted_at IS NULL;

-- name: Delete<Model> :execrows
DELETE FROM <schema>.<table>
WHERE id = $1;
```

## Phase 3: sqlc Configuration

Generate `sqlc.yaml` in the output directory:

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
```

## Phase 4: Atlas Migration Configuration

Generate `atlas.hcl` in the output directory:

```hcl
data "external_schema" "sqlc" {
  program = [
    "atlas-provider-sqlc",
    "load",
    "--dialect", "postgres",
    "--config", "file://sqlc.yaml",
  ]
}

env "local" {
  src = data.external_schema.sqlc.url
  dev = "docker://postgres/17/dev?search_path=public"
  migration {
    dir = "file://sql/migrations"
  }
}
```

Generate `sql/baseline.sql`:

```sql
-- This prevents Atlas from attempting to drop the public schema
CREATE SCHEMA IF NOT EXISTS public;
```

## Phase 5: Verify

- Confirm all generated files follow the directory layout:
  ```
  <output_dir>/
  ├── sql/
  │   ├── schema.sql
  │   ├── baseline.sql
  │   └── queries/
  │       └── <entity>.sql
  ├── sqlc.yaml
  └── atlas.hcl
  ```
- Verify that column types match the proto field types correctly
- Verify that entity columns (id, created_at, updated_at, deleted_at) are present in every table
- Present a summary of what was generated to the user

## Rules

- Always read the proto files before generating anything
- Preserve any existing sql files that are not entity-related (e.g. custom queries the user may have added)
- If schema.sql already exists, ask the user whether to regenerate or merge
- Use the exact naming conventions: snake_case for tables and columns, PascalCase for sqlc query names
- Do not generate Go code in this skill (that is handled by sqlc tooling and subsequent skills)
- If unsure about a type mapping, ask the user rather than guessing
