---
description: Generate Connect-RPC handler implementations backed by sqlc stores. Use when the user wants to scaffold or implement CRUD handlers for their proto services.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /api-handlers - Connect-RPC Handler Implementation

You are generating Connect-RPC handler implementations for proto services. Your goal is to read service definitions and sqlc-generated stores, then produce handler structs, proto-to-DB mappers, and RPC implementations.

Before generating, scan for existing code that overlaps with what this skill produces. If existing patterns are found, present divergences and ask the user to confirm a plan before proceeding. If no existing code is found, proceed directly. After generating, verify expected files exist with correct package declarations and imports, then present a summary. If target files already exist, ask the user before overwriting.

## Prerequisites

This skill requires artifacts from prior skills:
- **Service proto files** from `/protos` — `service_*.proto` and `rpc_*.proto` files defining RPCs and request/response types
- **sqlc-generated code** from `/db-schema` + `sqlc generate` — the `db/` package with `Queries`, model types, and query methods

If these do not exist, inform the user which skills to run first.

## Setup

1. Determine the source:
   - If the user provides a path (e.g. `/api-handlers protos/acme/inventory/v1/`), use it
   - Otherwise, search for `service_*.proto` files and ask the user which services to implement
2. Read the service proto files, sqlc-generated `db/` package (`models.go` and query files), and `go.mod`
3. Resolve paths from the proto `package` declaration:
   - Output directory for handlers: `internal/<provider>/<domain>/<version>/api/`
   - DB package: `internal/<provider>/<domain>/<version>/db/`
4. Derive Go import paths from the proto `go_package` option

## Generation

### Handler struct

Generate `api/handler_<entity_snake>.go` per entity service with:
- A `Deps` struct holding `*pgxpool.Pool`
- A private handler struct embedding `Unimplemented*ServiceHandler` and holding `*db.Queries`
- A public constructor returning `(string, http.Handler)` via the connect-generated `New*ServiceHandler`

### Mappers

Generate `api/mapper_<entity_snake>.go` with conversion functions between sqlc DB types and proto types:
- `<entity>ToProto` — DB row to proto message
- `<entity>FromCreate` — create request to sqlc params
- `<entity>FromUpdate` — update request to sqlc params

Derive field mappings from the actual proto message fields and sqlc-generated types in `db/models.go`. Convert snake_case to PascalCase respecting Go initialisms (`category_id` -> `CategoryID`).

### RPC implementations

Generate one file per RPC: `api/rpc_<operation>_<entity_snake>.go`. Only generate RPCs that have corresponding service definitions.

Follow these conventions:
- **Error codes**: `CodeAlreadyExists` for duplicate key (`pgconn.PgError` code `23505`), `CodeNotFound` for `pgx.ErrNoRows`, `CodeInvalidArgument` for bad input, `CodeInternal` for everything else
- **Create**: generate a UUID v4, set timestamps, call the store, handle duplicate key
- **Get**: parse UUID from request, call the store, handle not found
- **List**: cursor-based pagination using the store's paginated query, base64-encoded page tokens, default page size of 50
- **Update**: support partial updates via FieldMask — fetch current row, apply only masked fields from the request, preserve unmasked fields from the DB
- **Delete**: soft delete via the store, return `CodeNotFound` if no rows affected

### Validation

Do not generate a custom validation interceptor. Use `connectrpc.com/validate` as a handler option when wiring up the service:

```go
import "connectrpc.com/validate"

opts := []connect.HandlerOption{
    connect.WithInterceptors(validate.NewInterceptor()),
}
path, handler := New<Model>ServiceHandler(deps, opts...)
```

If `buf.validate` annotations are not present in the protos, skip — handler-level validation (UUID parsing, etc.) is sufficient.

## Rules

- Only generate RPC files for operations that have corresponding service RPCs
- RPC files (`api/rpc_*.go`) are customisable — do NOT overwrite existing ones
- Handler and mapper files are structural — may be regenerated
