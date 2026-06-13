---
description: Review the connect-ts-backend codebase for structural consistency, naming conventions, and cross-layer alignment. Use after running generation skills to verify everything holds together.
argument-hint: <output directory>
disable-model-invocation: true
---

# /review - Structural Review and Convention Compliance

You are reviewing a connect-ts-backend codebase to verify that all generated and hand-written code follows the expected structural patterns, naming conventions, and cross-layer alignment.

## Setup

1. Determine the review scope:
   - If the user provides a path (e.g. `/review src/acme/inventory/v1/`), review that module root
   - Otherwise, search for `src/*/*/v*/` directories and ask the user which to review
2. Read the proto files to build a reference list of entities, their fields, and configured operations
3. Scan all subdirectories (`schema/`, `queries/`, `handlers/`, `outbox/`, `workers/`, `consumers/`, `mcp/`) to understand which layers have been generated

## Check 1: Module Path Compliance

Verify the folder structure replicates the proto package:
- The proto `package` declaration (e.g. `acme.inventory.v1`) must map to `src/acme/inventory/v1/`
- All subdirectories must be direct children of this root
- TypeScript imports between sibling modules must use relative paths within this structure

Report any files that live outside this structure or use incorrect import paths.

## Check 2: File Naming Conventions

Verify every file follows the expected naming pattern:

| Layer | Expected files per entity |
|-------|--------------------------|
| `schema/` | `tables.ts` contains table definitions for every entity, `relations.ts` for foreign keys |
| `queries/` | `<entity_snake>.ts` per entity |
| `handlers/` | `handler_<entity_snake>.ts`, `mapper_<entity_snake>.ts`, `rpc_<operation>_<entity_snake>.ts` per operation |
| `outbox/` | `event_create_<entity_snake>.ts`, `event_update_<entity_snake>.ts`, `event_delete_<entity_snake>.ts` per entity |
| `workers/` | `envelope.ts`, `register_<entity_snake>.ts`, `worker_<operation>_<entity_snake>.ts` per operation |
| `consumers/` | `consumer_<subscriber>_<entity_snake>.ts` per subscriber |
| `mcp/` | `registry_<entity_snake>.ts`, `tool_<operation>_<entity_snake>.ts` per operation |
| Proto dir | `service_<entity_snake>.proto`, `rpc_<operation>_<entity_snake>.proto` per operation |

Report any missing files, misnamed files, or unexpected files.

## Check 3: Naming Convention Consistency

Verify naming consistency across all layers:
- **Entity names**: PascalCase in TypeScript and proto (`Product`), snake_case in file names and DB columns (`product`)
- **Table names**: schema-qualified in Drizzle (`acme_inventory_v1.product`)
- **Query function names**: `get<Model>`, `list<Model>s`, `list<Model>sPaginated`, `create<Model>`, `update<Model>`, `softDelete<Model>`, `delete<Model>`
- **Handler factory names**: `create<Model>ServiceHandler`
- **Mapper functions**: `<entity>ToProto`, `<entity>FromCreate`, `<entity>FromUpdate`
- **Outbox event types**: `create_<model_snake>`, `update_<model_snake>`, `delete_<model_snake>`
- **Kafka topics**: `<domain>.<model_snake>.events.<version>`
- **Consumer group IDs**: `<domain>.<model_snake>.<subscriber_lower>`
- **MCP tool names**: `create_<model_snake>`, `get_<model_snake>`, `list_<model_snake>s`, `update_<model_snake>`, `delete_<model_snake>`

## Check 4: Cross-Layer Alignment

Verify layers are consistent with each other:
- **Proto to Schema**: every entity field has a corresponding Drizzle column with correct type
- **Schema to Queries**: query helpers exist for every table, column references match schema
- **Queries to Handlers**: mapper functions reference correct Drizzle and proto types, handler deps include `db`
- **Handlers to Outbox**: if outbox is present, mutating RPCs use transactions, outbox event types match
- **Outbox to Workers**: worker types reference correct outbox events, topic constants follow convention
- **Workers to Consumers**: consumer topics match worker constants, consumer group IDs follow convention
- **Handlers to MCP**: MCP registry references Connect-ES service interface, tool methods call correct RPCs

## Check 5: Validation Coverage

Verify proto validation annotations:
- All `string id` fields have `[(buf.validate.field).string.uuid = true]`
- `page_size` fields have bounded range constraints
- `repeated` fields on request messages have `max_items`
- `item` fields on Create/Update requests have `required = true`
- Enum fields have `defined_only = true`

### Server-side enforcement
- If `buf.validate` annotations are present, verify that handlers use `@bufbuild/protovalidate` Connect interceptor
- Flag any custom validation middleware — the official `@bufbuild/protovalidate` interceptor should be used instead

Report unbounded repeated fields as **warnings**.

## Check 6: Structural Completeness

For each entity, verify that all adopted layers are complete:
- If `schema/` exists, verify tables + queries + drizzle.config.ts
- If `handlers/` exists, verify handler + mapper + all RPC files for configured operations
- If `outbox/` exists, verify event definitions for all mutating operations AND that handlers use transactions
- If `workers/` exists, verify envelope + registration + worker per mutating operation
- If `consumers/` exists, verify at least one consumer per entity
- If `mcp/` exists, verify registry + tool per configured operation

Report entities with partial coverage.

## Output Format

Present findings as a structured report with: Summary, Issues (location, category, severity, description, fix), and Recommendations.

## Rules

- Read all relevant files before reporting — do not flag issues based on assumptions
- Distinguish errors (will not compile) from warnings (convention violations)
- Do not modify any files — this skill is read-only
- Only review layers that are present
- Be specific with file paths and line numbers
