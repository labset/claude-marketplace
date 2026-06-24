---
description: Verify the connect-backend codebase for structural consistency, naming conventions, and cross-layer alignment. Use after running generation skills to verify everything holds together.
argument-hint: <output directory>
disable-model-invocation: true
---

# /verify - Structural Verification and Convention Compliance

You are reviewing a connect-backend codebase to verify that all generated and hand-written code follows the expected structural patterns, naming conventions, and cross-layer alignment. Your goal is to identify gaps, inconsistencies, and convention violations — then suggest targeted fixes.

## Setup

1. Determine the review scope:
   - If the user provides a path (e.g. `/verify internal/acme/inventory/v1/`), review that package root
   - Otherwise, search for `internal/*/*/v*/` directories and ask the user which to review
2. Read the proto files to build a reference list of entities, their fields, and configured operations
3. Scan all subpackages (`sql/`, `db/`, `api/`, `outbox/`, `workers/`, `consumers/`, `mcp/`) to understand which layers have been generated

## Check 1: Package Path Compliance

Verify the folder structure replicates the proto package:

- The proto `package` declaration (e.g. `acme.inventory.v1`) must map to the folder path `internal/acme/inventory/v1/`
- All subpackages must be direct children of this root
- Go `package` declarations in each subpackage must match the folder name (`package api`, `package db`, `package outbox`, etc.)
- Go imports between sibling subpackages must use `<module>/internal/<provider>/<domain>/<version>/<subpackage>`

Report any files that live outside this structure or use incorrect import paths.

## Check 2: File Naming Conventions

Verify every file follows the expected naming pattern:

| Layer | Expected files per entity |
|-------|--------------------------|
| `sql/schema.sql` | Contains `CREATE TABLE` for every entity |
| `sql/queries/` | `<entity_snake>.sql` per entity |
| `api/` | `handler_<entity_snake>.go`, `mapper_<entity_snake>.go`, `rpc_<operation>_<entity_snake>.go` per operation |
| `outbox/` | `event_create_<entity_snake>.go`, `event_update_<entity_snake>.go`, `event_delete_<entity_snake>.go` per entity |
| `workers/` | `envelope.go`, `register_<entity_snake>.go`, `worker_<operation>_<entity_snake>.go` per operation |
| `consumers/` | `consumer_<subscriber>_<entity_snake>.go` per subscriber |
| `mcp/` | `registry_<entity_snake>.go`, `tool_<operation>_<entity_snake>.go` per operation |
| Proto dir | `service_<entity_snake>.proto`, `rpc_<operation>_<entity_snake>.proto` per operation |

Report any missing files, misnamed files, or unexpected files that don't match the convention.

## Check 3: Naming Convention Consistency

Verify naming consistency across all layers:

- **Entity names**: PascalCase in Go and proto (`Product`), snake_case in SQL and file names (`product`)
- **Table names**: `<schema_name>.<entity_snake>` (e.g. `acme_inventory_v1.product`)
- **Schema name**: underscore-joined proto package segments (e.g. `acme_inventory_v1`)
- **sqlc query names**: `Get<Model>`, `List<Model>s`, `List<Model>sPaginated`, `Create<Model>`, `Update<Model>`, `SoftDelete<Model>`, `Delete<Model>`
- **Handler types**: `<modelLower>Handler` (e.g. `productHandler`)
- **Deps types**: `<Model>Deps` (e.g. `ProductDeps`)
- **Constructor names**: `New<Model>ServiceHandler`
- **Mapper functions**: `<modelLower>ToProto`, `<modelLower>FromCreate`, `<modelLower>FromUpdate`
- **Outbox event kinds**: `create_<model_snake>`, `update_<model_snake>`, `delete_<model_snake>`
- **Kafka topics**: `<domain>.<model_snake>.events.<version>`
- **Consumer group IDs**: `<domain>.<model_snake>.<subscriber_lower>`
- **MCP tool names**: `create_<model_snake>`, `get_<model_snake>`, `list_<model_snake>s`, `update_<model_snake>`, `delete_<model_snake>`

Report any naming that deviates from these conventions.

## Check 4: Cross-Layer Alignment

Verify that layers are consistent with each other:

### Proto to Schema
- Every entity message field must have a corresponding column in `schema.sql`
- Column types must match the proto-to-PostgreSQL type mapping
- Entity columns (`id`, `created_at`, `updated_at`, `deleted_at`) must be present in every table
- Reference fields must have `UUID` columns with appropriate foreign key constraints

### Schema to sqlc
- `sqlc.yaml` must reference the correct schema and query paths
- Query files must exist for every table in `schema.sql`
- Query column lists must match the schema (no missing columns in INSERT/UPDATE)

### sqlc to Handlers
- Mapper functions must reference the correct sqlc-generated types from `db/models.go`
- SQLC field names (PascalCase with Go initialisms) must match what mappers use
- Handler `store` field type must match the sqlc `Queries` type

### Handlers to Outbox
- If outbox is present, mutating RPCs must use transactions (`h.pool.Begin`, `h.store.WithTx`, `h.river.InsertTx`)
- Outbox event arg types must match what RPC implementations insert
- Handler `Deps` struct must include `Pool` and `River` fields
- Non-mutating RPCs (Get, List) must NOT use transactions

### Outbox to Workers
- Worker types must reference the correct outbox event args
- `Register<Model>Workers` must register a worker for every outbox event type
- Topic constants must follow the naming convention

### Workers to Consumers
- Consumer topics must match worker topic constants
- Consumer group IDs must follow the naming convention
- Consumer handler interfaces must accept `workers.EventEnvelope`

### Handlers to MCP
- MCP registry must reference the correct Connect service handler interface
- Tool methods must call the correct handler RPC methods
- Tool names must match the operations available in the service

## Check 5: Validation Coverage

Verify that proto validation annotations are properly applied:

### Request-level validation
- All `string id` fields on request messages must have `[(buf.validate.field).string.uuid = true]`
- `page_size` fields must have bounded range constraints (e.g. `int32 = {gte: 0, lte: 100}`)
- `item` fields on Create and Update requests should have `[(buf.validate.field).required = true]`

### Repeated field bounds
- All `repeated` fields on **request messages** must have `(buf.validate.field).repeated.max_items` — flag any that are unbounded
- All `repeated` fields on **entity model messages** should have `max_items` constraints — flag any that are unbounded and ask whether a limit should be added
- `FieldMask` paths are implicitly repeated — check whether update masks are bounded

### String field bounds
- String fields that represent user input (names, descriptions, URLs, etc.) should have `max_len` constraints
- Flag string fields without `max_len` as informational — not every string needs a bound, but the user should consciously decide

### Enum validation
- Enum fields should have `(buf.validate.field).enum.defined_only = true` to reject unknown values
- Flag enum fields without this annotation

### Server-side enforcement
- If `buf.validate` annotations are present, verify that handler constructors use `connectrpc.com/validate` interceptor (e.g. `validate.NewInterceptor()` in handler options)
- Flag any custom validation interceptor (`interceptor_validate.go` or `protovalidate` usage) — the official `connectrpc.com/validate` package should be used instead

Report unbounded repeated fields as **warnings** (convention violation that could enable abuse).

## Check 6: Structural Completeness

For each entity, verify that all adopted layers are complete:

- If `sql/` exists, verify schema + queries + sqlc.yaml + atlas.hcl
- If `api/` exists, verify handler + mapper + all RPC files for configured operations
- If `outbox/` exists, verify event args for all mutating operations AND that handlers use transactions
- If `workers/` exists, verify envelope + registration + worker per mutating operation
- If `consumers/` exists, verify at least one consumer per entity
- If `mcp/` exists, verify registry + tool per configured operation

Report entities that have partial coverage (e.g. handler exists but mapper is missing).

## Output Format

Present findings as a structured report:

### Summary
- Entities reviewed: `<list>`
- Layers present: `<list>`
- Overall status: `<clean / issues found>`

### Issues

For each issue found:
- **Location**: file path and line number
- **Category**: naming / structure / cross-layer / completeness
- **Severity**: error (will break) / warning (convention violation) / info (suggestion)
- **Description**: what is wrong
- **Fix**: specific action to resolve it

### Recommendations

If the review surfaces patterns that should be addressed across the board (e.g. "all mappers are missing enum handling"), group them as recommendations rather than per-file issues.

## Rules

- This skill is read-only — do not modify any files
- Distinguish between errors (will not compile) and warnings (convention violations)
- Only review layers that are present — do not flag missing optional layers
- Be specific with file paths and line numbers
