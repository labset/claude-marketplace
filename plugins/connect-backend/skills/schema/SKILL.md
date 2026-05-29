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
   - An entity message is one that embeds a `labset.data.v1.Entity` field, or has `ROLE_ENTITY` in its options, or has fields like `id`, `created_at`, `updated_at` that indicate it is a database-backed entity
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

## Codebase Assessment

Before generating anything, scan the existing codebase to understand what already exists and identify divergences from the target conventions. Present your findings to the user before proceeding.

### What to look for

1. **Existing database layer**:
   - Search for existing SQL files (`*.sql`), migration directories, or ORM usage (e.g. GORM, Ent, sqlboiler)
   - Check for existing `sqlc.yaml` or `sqlc.json` configuration
   - Check for existing Atlas, Flyway, golang-migrate, or goose migration setups
   - Look for existing database connection code to understand the driver in use (pgx, database/sql, etc.)

2. **Existing package layout**:
   - Check whether the project uses `internal/` or a flat layout
   - Check whether existing Go packages follow the proto package path convention (`internal/<provider>/<domain>/<version>/`)
   - If not, identify the current layout and how it diverges

3. **Existing schema definitions**:
   - Look for existing CREATE TABLE statements, model structs, or schema definitions
   - Check whether tables use the expected entity columns (id UUID, created_at, updated_at, deleted_at)
   - Identify any existing tables that correspond to proto messages

### What to present to the user

Summarise your findings as a short assessment:
- **Matches**: aspects of the existing codebase that already align with the target conventions
- **Divergences**: aspects that differ — different package layout, different migration tool, missing soft-delete columns, different naming conventions, etc.
- **Proposed plan**: for each divergence, suggest one of:
  - **Adopt as-is**: the existing pattern is fine and the skill should adapt to it (e.g. the project uses `golang-migrate` instead of Atlas — keep it)
  - **Incremental refactor**: suggest a specific, scoped change to move toward the convention (e.g. "add `deleted_at` column to existing tables for soft-delete support")
  - **Generate alongside**: generate the new code in the target layout and let existing code coexist until the user is ready to migrate

Ask the user to confirm the plan before proceeding to generation. If the codebase is greenfield (no existing database layer), skip the assessment and proceed directly.

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
| `enum` | `TEXT` |
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

Each query file must use sqlc annotations with **named parameters** (`@param` style) and **explicit column lists** (not `SELECT *`). Generate the following queries:

```sql
-- name: Get<Model> :one
SELECT <all columns>
FROM <schema>.<table>
WHERE id = @id AND deleted_at IS NULL;

-- name: List<Model>s :many
SELECT <all columns>
FROM <schema>.<table>
WHERE deleted_at IS NULL
ORDER BY created_at DESC;

-- name: List<Model>sPaginated :many
SELECT <all columns>
FROM <schema>.<table>
WHERE deleted_at IS NULL
  AND (@cursor::uuid = '00000000-0000-0000-0000-000000000000'::uuid OR id > @cursor)
ORDER BY id
LIMIT @page_size;

-- name: Create<Model> :one
INSERT INTO <schema>.<table> (<all columns except deleted_at>)
VALUES (@id, @created_at, @updated_at, <@named params for each user field>)
RETURNING *;

-- name: Update<Model> :one
UPDATE <schema>.<table>
SET <all user fields as @named params>, updated_at = @updated_at
WHERE id = @id AND deleted_at IS NULL
RETURNING *;

-- name: SoftDelete<Model> :execrows
UPDATE <schema>.<table>
SET deleted_at = NOW()
WHERE id = @id AND deleted_at IS NULL;

-- name: Delete<Model> :exec
DELETE FROM <schema>.<table>
WHERE id = @id;
```

The `List<Model>sPaginated` query uses a zero UUID sentinel for the initial page — when `@cursor` is the zero UUID, the `OR id > @cursor` condition returns all rows. Subsequent pages pass the last row's ID as the cursor, returning rows with `id > @cursor` in ascending order.

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
env "local" {
  src = [
    "file://sql/baseline.sql",
    "file://sql/schema.sql",
  ]
  dev = "docker://postgres/17-alpine/dev"
  migration {
    dir = "file://migrations"
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
  ├── migrations/
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
