---
description: Generate Connect-RPC handler implementations backed by Drizzle ORM queries. Use when the user wants to scaffold or implement CRUD handlers for their proto services.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /handlers - Connect-RPC Handler Implementation

You are generating TypeScript Connect-RPC handler implementations for proto services. Your goal is to read service definitions and Drizzle query helpers, then produce handler modules, proto-to-DB mappers, and RPC implementations.

## Prerequisites

This skill requires artifacts from prior skills:
- **Service proto files** from `/service` — defining RPCs and request/response types
- **Generated TypeScript** from `buf generate` — Connect-ES service stubs and Protobuf-ES message types
- **Drizzle query helpers** from `/schema` — the `queries/` module with typed database operations

If these do not exist, inform the user which skills to run first.

## Setup

1. Determine the source:
   - If the user provides a path (e.g. `/handlers protos/acme/inventory/v1/`), use it
   - Otherwise, search for `service_*.proto` files and ask the user which services to implement
2. Read the service proto files, generated TypeScript stubs, and Drizzle query helpers
3. Resolve paths from the proto `package` declaration:
   - Output directory for handlers: `src/<provider>/<domain>/<version>/handlers/`
   - Queries module: `src/<provider>/<domain>/<version>/queries/`
4. Derive import paths from the generated Connect-ES output

## Codebase Assessment

Before generating, scan for existing handler patterns, data access layer, and module layout. If existing code is found, present divergences and a proposed plan (adopt existing patterns, suggest incremental refactors, or generate alongside). Ask the user to confirm before proceeding. If no existing handler code exists, skip and proceed directly.

## Generation

### Handler module

Generate `handlers/handler_<entity_snake>.ts` per entity service with:
- A `Deps` type holding the Drizzle `db` instance (and any other dependencies)
- A factory function that returns a Connect service implementation object
- The factory uses the generated Connect-ES `ServiceType` for type safety

### Mappers

Generate `handlers/mapper_<entity_snake>.ts` with conversion functions between Drizzle row types and Protobuf-ES message types:
- `<entity>ToProto` — DB row to proto message instance
- `<entity>FromCreate` — create request to Drizzle insert values
- `<entity>FromUpdate` — update request to Drizzle update values

Derive field mappings from the actual proto message types and Drizzle schema.

### RPC implementations

Generate one file per RPC: `handlers/rpc_<operation>_<entity_snake>.ts`. Only generate RPCs that have corresponding service definitions.

Follow these conventions:
- **Error codes**: `ConnectError` with `Code.AlreadyExists` for duplicate key, `Code.NotFound` for missing rows, `Code.InvalidArgument` for bad input, `Code.Internal` for everything else
- **Create**: generate a UUID v4, set timestamps, call the query helper, handle duplicate key (catch Postgres error code `23505`)
- **Get**: parse UUID from request, call the query helper, handle not found
- **List**: cursor-based pagination using the query helper's paginated function, base64-encoded page tokens, default page size of 50
- **Update**: support partial updates via FieldMask — fetch current row, apply only masked fields from the request, preserve unmasked fields from the DB
- **Delete**: soft delete via the query helper, return `Code.NotFound` if no rows affected

### Validation

Do not generate custom validation middleware. Use `@connectrpc/connect` with `@bufbuild/protovalidate` interceptor for server-side validation:

```typescript
import { createValidateInterceptor } from "@bufbuild/protovalidate/connect";

const interceptors = [createValidateInterceptor()];
```

If `buf.validate` annotations are not present in the protos, skip — handler-level validation (UUID parsing, etc.) is sufficient.

## Verify

- Confirm generated files follow the layout: `handler_*.ts`, `mapper_*.ts`, `rpc_*_*.ts` under `handlers/`
- Verify imports resolve against `package.json` and the generated Connect-ES output
- Verify mapper functions reference the correct Drizzle and proto types
- Present a summary to the user

## Rules

- Always read the Drizzle queries and service protos before generating to ensure type names match
- Only generate RPC files for operations that have corresponding service RPCs
- If a `handlers/rpc_*.ts` file already exists, do NOT overwrite it (it may contain user customizations)
- Handler and mapper files may be regenerated (they are structural, not customized)
