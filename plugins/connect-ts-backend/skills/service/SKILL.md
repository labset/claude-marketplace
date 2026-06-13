---
description: Generate Connect-RPC service proto definitions from entity messages. Use when the user wants to create service RPCs, request/response types for their proto entities.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /service - Connect-RPC Service Proto Generation

You are generating Connect-RPC service `.proto` files from entity message definitions. Your goal is to read proto messages and produce service definitions with RPCs, request/response types, and proper imports.

## Setup

1. Determine the proto source:
   - If the user provides a path (e.g. `/service protos/acme/inventory/v1/`), use it
   - Otherwise, search for `.proto` files and ask the user which entities to generate services for
2. Read the proto files and identify entity messages (same criteria as `/schema`)
3. Ask the user which CRUD operations each entity needs (default: all five)
4. Resolve the package path and detect `buf.gen.yaml` for TypeScript codegen configuration
   - Verify `@connectrpc/protoc-gen-connect-es` and `@bufbuild/protoc-gen-es` are configured

## Codebase Assessment

Before generating, scan for existing service definitions, proto file organisation, and RPC patterns. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing service protos exist, skip and proceed directly.

## Service Definition

For each entity, generate `service_<entity_snake>.proto` with a `service <Model>Service` block containing only the selected operations:
- `Create<Model>`, `Get<Model>`, `List<Model>s`, `Update<Model>`, `Delete<Model>`

Use the same `package` as existing proto files.

## Request/Response Types

Generate one file per RPC: `rpc_<operation>_<entity_snake>.proto`. Each file gets its own `syntax`, `package`, and imports.

Follow these patterns:
- **Create**: `item` field with `[(buf.validate.field).required = true]`
- **Get/Delete**: `string id` with `[(buf.validate.field).string.uuid = true]`
- **List**: `int32 page_size` with `[(buf.validate.field).int32 = {gte: 0, lte: 100}]`, `string page_token`, response has `repeated <Model> items` and `string next_page_token`
- **Update**: `string id` (uuid-validated), `item` (required), `google.protobuf.FieldMask update_mask`

All RPC protos must import `buf/validate/validate.proto` for validation annotations.

## Entity Model Validation

After generating RPC protos, review entity model messages for fields that need validation constraints:
- `repeated` fields: add `max_items` — ask the user for appropriate limits
- `string` fields with known bounds: add `max_len`
- `enum` fields: add `defined_only = true`

Present proposed annotations to the user for confirmation before modifying model protos.

## Verify

- Confirm files follow naming: `service_<entity>.proto`, `rpc_<operation>_<entity>.proto`
- Verify import paths and `package` match existing proto files
- Verify `buf.gen.yaml` includes TypeScript plugins for Connect-ES and Protobuf-ES
- Verify all `id` fields have UUID validation, `page_size` has bounded range, `repeated` request fields have `max_items`
- Present a summary to the user

## Rules

- Always read existing proto files to extract correct `package`
- Do not modify existing proto files without presenting proposed changes to the user first
- If a service proto already exists, ask the user whether to regenerate or skip
- Always use `buf/validate/validate.proto` for input validation
- Never leave `repeated` fields on request messages unbounded
- Each RPC message type goes in its own file to keep protos modular
- If the project does not yet depend on `buf/validate`, inform the user to add it to their `buf.yaml` deps
