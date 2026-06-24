---
description: Generate Connect-RPC proto definitions (entity models, service RPCs, request/response types) from entity messages. Use when the user wants to create or update proto files for their entities.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /protos - Connect-RPC Proto Generation

You are generating Connect-RPC service `.proto` files from entity message definitions. Your goal is to read proto messages and produce service definitions with RPCs, request/response types, and proper imports.

See `CONVENTIONS.md` for shared conventions (codebase assessment, verify step, overwrite protection).

## Setup

1. Determine the proto source:
   - If the user provides a path (e.g. `/protos protos/acme/inventory/v1/`), use it
   - Otherwise, search for `.proto` files and ask the user which entities to generate services for
2. Read the proto files and identify entity messages (same criteria as `/db-schema`)
3. Ask the user which CRUD operations each entity needs (default: all five)
4. Resolve the package path and reuse the `go_package` option from existing proto files

## Service Definition

For each entity, generate `service_<entity_snake>.proto` with a `service <Model>Service` block containing only the selected operations:
- `Create<Model>`, `Get<Model>`, `List<Model>s`, `Update<Model>`, `Delete<Model>`

Use the same `package` and `go_package` as existing proto files.

## Request/Response Types

Generate one file per RPC: `rpc_<operation>_<entity_snake>.proto`. Each file gets its own `syntax`, `package`, `go_package`, and imports.

Follow these patterns:
- **Create**: `item` field with `[(buf.validate.field).required = true]`
- **Get/Delete**: `string id` with `[(buf.validate.field).string.uuid = true]`
- **List**: `int32 page_size` with `[(buf.validate.field).int32 = {gte: 0, lte: 100}]`, `string page_token`, response has `repeated <Model> items` and `string next_page_token`
- **Update**: `string id` (uuid-validated), `item` (required), `google.protobuf.FieldMask update_mask`

All RPC protos must import `buf/validate/validate.proto` for validation annotations.

## Entity Model Validation

After generating RPC protos, review entity model messages for fields that need validation constraints:
- `repeated` fields: add `max_items` — ask the user for appropriate limits
- `string` fields with known bounds (names, descriptions, URLs): add `max_len`
- `enum` fields: add `defined_only = true`

Present proposed annotations to the user for confirmation before modifying model protos.

## Rules

- Do not modify existing proto files without presenting proposed changes to the user first
- Always use `buf/validate/validate.proto` for input validation — if not in `buf.yaml` deps, inform the user
- Never leave `repeated` fields on request messages unbounded
- Each RPC message type goes in its own file to keep protos modular
