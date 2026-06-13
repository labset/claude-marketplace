---
description: Generate Drizzle ORM schema, queries, and migration config from proto message definitions. Use when the user wants to create or update the database layer for their proto entities.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /schema - Database Schema and Query Generation

You are generating the database layer for a TypeScript Connect-RPC backend. Your goal is to read proto message definitions and produce Drizzle ORM schema definitions, typed query helpers, and migration configuration.

## Setup

1. Determine the proto source:
   - If the user provides a path (e.g. `/schema protos/acme/inventory/v1/`), use it
   - Otherwise, search for `.proto` files and ask the user which messages to target
2. Read the proto files and identify entity messages (messages with `id`, `created_at`, `updated_at` fields, or `ROLE_ENTITY` options, or `labset.data.v1.Entity` embedding)
3. Resolve the package path from the proto `package` declaration:
   - `acme.inventory.v1` becomes `src/acme/inventory/v1/`
   - All subdirectories (`schema/`, `queries/`, `handlers/`, `outbox/`, `workers/`, `consumers/`, `mcp/`) nest under this path
4. Read `package.json` for the project name and detect the package manager (npm, pnpm, bun, yarn)

## Codebase Assessment

Before generating, scan for existing database code, ORM usage, and migration tooling. If existing patterns are found, present divergences and a proposed plan (adopt, refactor incrementally, or generate alongside). Ask the user to confirm before proceeding. If greenfield, skip and proceed directly.

## Schema Generation

Generate `schema/tables.ts` in the output directory.

- Derive schema name from proto package: `acme.inventory.v1` becomes `acme_inventory_v1`
- Use `pgSchema` from `drizzle-orm/pg-core` for namespaced tables

Each entity table includes:
- **Entity columns**: `id uuid PRIMARY KEY` (with `defaultRandom()`), `createdAt timestamp NOT NULL` (with `defaultNow()`), `updatedAt timestamp NOT NULL` (with `defaultNow()`), `deletedAt timestamp` (nullable, for soft deletes)
- **User-defined fields** mapped from proto types:

| Proto Type | Drizzle Column |
|---|---|
| `string` | `text()` |
| `bytes` | `bytea()` (custom column) |
| `bool` | `boolean()` |
| `int32`, `sint32`, `sfixed32`, `uint32`, `fixed32` | `integer()` |
| `int64`, `sint64`, `sfixed64`, `uint64`, `fixed64` | `bigint({ mode: 'number' })` |
| `float`, `double` | `doublePrecision()` |
| `google.protobuf.Timestamp` | `timestamp({ withTimezone: true })` |
| `google.protobuf.Duration` | `interval()` |
| `google.protobuf.Struct` / `Value` | `jsonb()` |
| `enum` | `text()` |
| `repeated <scalar>` | array column (e.g. `text().array()`) |
| `repeated <message>`, nested message, `map` | `jsonb()` |
| `oneof` variants | nullable columns, one per variant |
| Reference fields (messages ending in `Ref`) | `uuid()` with `.references()` |

All columns are `.notNull()` by default except `deletedAt`, `oneof` variants, and wrapper types.

Generate `schema/relations.ts` with Drizzle relation definitions for any foreign key references between entities.

## Query Generation

Generate `queries/<entity_snake>.ts` per entity with typed query helpers using Drizzle's query builder:

- `get<Model>` — by `id`, filtered by `isNull(deletedAt)`
- `list<Model>s` — ordered by `desc(createdAt)`, filtered by `isNull(deletedAt)`
- `list<Model>sPaginated` — cursor-based using `gt(id, cursor)` with limit, ordered by `asc(id)`
- `create<Model>` — insert returning all columns
- `update<Model>` — update by `id` returning all columns, filtered by `isNull(deletedAt)`
- `softDelete<Model>` — sets `deletedAt` to `new Date()`, filtered by `isNull(deletedAt)`
- `delete<Model>` — hard delete

Each query function takes a Drizzle `db` instance (or transaction) as its first argument for composability.

## Drizzle Configuration

Generate `drizzle.config.ts` in the output directory:

```typescript
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./schema/tables.ts",
  out: "./migrations",
  dialect: "postgresql",
});
```

## Verify

- Confirm layout: `schema/tables.ts`, `schema/relations.ts`, `queries/<entity>.ts`, `drizzle.config.ts`
- Verify column types match proto field types
- Verify entity columns are present in every table
- Present a summary to the user

## Rules

- Always read the proto files before generating
- Preserve existing schema files that are not entity-related
- If schema files already exist, ask the user whether to regenerate or merge
- Use camelCase for TypeScript column names, snake_case for database column names (use Drizzle's column name mapping)
- Do not generate raw SQL — use Drizzle ORM throughout
