---
description: Generate Connect-RPC handler implementations backed by sqlc stores. Use when the user wants to scaffold or implement CRUD handlers for their proto services.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /handlers - Connect-RPC Handler Implementation

You are generating Connect-RPC handler implementations for proto services. Your goal is to read service definitions and sqlc-generated stores, then produce handler structs, proto-to-DB mappers, and RPC implementations.

## Setup

1. Determine the source:
   - If the user provides a path (e.g. `/handlers protos/acme/inventory/v1/`), use it
   - Otherwise, search for `service_*.proto` files and ask the user which services to implement
2. Read the service proto files to understand available RPCs
3. Read the sqlc-generated code in the `db/` package within the output directory to understand available queries and types
4. Read `go.mod` to determine the Go module path
5. Resolve the package path from the proto file's `package` declaration:
   - The proto package segments map directly to the folder path: `acme.inventory.v1` becomes `internal/acme/inventory/v1/`
   - The output directory for handlers is `internal/<provider>/<domain>/<version>/api/`
   - The `db/` package produced by `/schema` lives at `internal/<provider>/<domain>/<version>/db/`
   - All Go imports for sibling subpackages use `<module>/internal/<provider>/<domain>/<version>/<subpackage>`
6. Derive import paths from the proto file's `go_package` option:
   - **protoImport**: the full Go import path (e.g. `github.com/acme/inventory/v1`)
   - **protoAlias**: the package alias (e.g. `inventoryv1`)
   - **connectImport**: `<protoImport>/<protoAlias>connect` (e.g. `github.com/acme/inventory/v1/inventoryv1connect`)
   - **connectAlias**: `<protoAlias>connect`

## Codebase Assessment

Before generating anything, scan the existing codebase to understand what already exists and identify divergences from the target conventions. Present your findings to the user before proceeding.

### What to look for

1. **Existing handler implementations**:
   - Search for existing Connect-RPC, gRPC, or HTTP handler code
   - Check whether handlers use the `Deps` struct + constructor pattern or a different wiring approach (e.g. direct struct init, dependency injection framework)
   - Check whether handlers embed `Unimplemented*ServiceHandler` or use a different base
   - Look for existing RPC method implementations and their error handling patterns

2. **Existing data access layer**:
   - Check whether the project uses sqlc-generated stores, raw SQL, an ORM, or a repository pattern
   - Look for existing mapper/converter functions between database types and proto types
   - Check whether the `db/` package follows sqlc conventions or uses a different structure
   - Identify the `Queries` type name and any schema prefixes on generated types (read `db/models.go` if it exists)

3. **Existing package layout**:
   - Check whether handler code lives in `api/`, `handler/`, `server/`, `service/`, or another location
   - Check whether there is one handler file per entity or a monolithic handler
   - Check file naming: `handler_<entity>.go` vs `<entity>_handler.go` vs other patterns

4. **Existing error handling**:
   - Check whether existing code uses `connect.NewError` with status codes or a different error approach
   - Look for existing duplicate-key detection (pgconn error code `23505`) or not-found handling

### What to present to the user

Summarise your findings as a short assessment:
- **Matches**: handler patterns that already align (e.g. already uses `Deps` + constructor, already embeds `Unimplemented`)
- **Divergences**: different handler wiring, different package location, different error patterns, ORM instead of sqlc, etc.
- **Proposed plan**: for each divergence, suggest one of:
  - **Adopt as-is**: follow the project's existing handler pattern (e.g. handlers live in `server/` — generate there instead of `api/`)
  - **Incremental refactor**: suggest specific changes (e.g. "extract handler per entity from monolithic handler", "add `Deps` struct for dependency injection")
  - **Generate alongside**: generate new handlers in `api/` alongside existing code, letting the user migrate at their own pace

Ask the user to confirm the plan before proceeding to generation. If no existing handler code exists, skip the assessment and proceed directly.

## Phase 1: Handler Struct

Generate `api/handler_<entity_snake>.go` for each entity service:

```go
package api

import (
    "net/http"

    "connectrpc.com/connect"
    "github.com/jackc/pgx/v5/pgxpool"

    "<module>/db"
    <protoAlias> "<protoImport>"
    <connectAlias> "<connectImport>"
)

type <Model>Deps struct {
    Pool *pgxpool.Pool
}

type <modelLower>Handler struct {
    <connectAlias>.Unimplemented<Model>ServiceHandler
    store *db.Queries
}

func New<Model>ServiceHandler(deps <Model>Deps, opts ...connect.HandlerOption) (string, http.Handler) {
    h := &<modelLower>Handler{
        store: db.New(deps.Pool),
    }
    return <connectAlias>.New<Model>ServiceHandler(h, opts...)
}
```

Where `<modelLower>` is the model name with the first letter lowercased (e.g. `Product` becomes `productHandler`).

## Phase 2: Mapper Functions

Generate `api/mapper_<entity_snake>.go` with proto-to-DB and DB-to-proto conversion functions:

```go
package api

import (
    "github.com/gofrs/uuid/v5"
    "github.com/jackc/pgx/v5/pgtype"
    "google.golang.org/protobuf/types/known/timestamppb"

    pluginv1 "path/to/clarity/plugin/v1"
    "<module>/db"
    <protoAlias> "<protoImport>"
)

// <modelLower>ToProto converts a database row to a proto message.
func <modelLower>ToProto(row db.<DBPrefix><Model>) *<protoAlias>.<Model> {
    return &<protoAlias>.<Model>{
        Entity: &pluginv1.Entity{
            Id:        row.ID.String(),
            CreatedAt: timestamppb.New(row.CreatedAt.Time),
            UpdatedAt: timestamppb.New(row.UpdatedAt.Time),
        },
        // Map each field:
        // - Regular fields: row.<SQLCName>
        // - Ref fields: &<RefType>{Id: row.<FieldName>ID.String()}
        // - Enum fields: <EnumType>(<EnumType>_value[row.<SQLCName>])
    }
}

// <modelLower>FromCreate converts a create request to database params.
func <modelLower>FromCreate(msg *<protoAlias>.<Model>, id uuid.UUID, now pgtype.Timestamptz) db.Create<Model>Params {
    return db.Create<Model>Params{
        ID:        id,
        CreatedAt: now,
        UpdatedAt: now,
        // Map each user field from msg
    }
}

// <modelLower>FromUpdate converts an update request to database params.
func <modelLower>FromUpdate(msg *<protoAlias>.<Model>, id uuid.UUID) db.Update<Model>Params {
    return db.Update<Model>Params{
        ID: id,
        // Map each user field from msg
    }
}
```

### SQLC name convention

When referencing sqlc-generated field names, convert snake_case to PascalCase respecting Go initialisms:
- `category_id` becomes `CategoryID` (not `CategoryId`)
- `name` becomes `Name`
- `http_url` becomes `HTTPURL`

### DB prefix

The sqlc-generated types may have a prefix derived from the schema name. Check the actual sqlc output in `db/models.go` to determine the exact type names (e.g. `db.AcmeInventoryV1Product` or just `db.Product`).

## Phase 3: RPC Implementations

Generate one file per RPC: `api/rpc_<operation>_<entity_snake>.go`

### Create (`rpc_create_<entity>.go`)

```go
func (h *<modelLower>Handler) Create<Model>(
    ctx context.Context,
    req *connect.Request[<protoAlias>.Create<Model>Request],
) (*connect.Response[<protoAlias>.Create<Model>Response], error) {
    id, err := uuid.NewV4()
    if err != nil {
        return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("generate id: %w", err))
    }
    now := pgtype.Timestamptz{Time: time.Now(), Valid: true}
    params := <modelLower>FromCreate(req.Msg.GetItem(), id, now)

    row, err := h.store.Create<Model>(ctx, params)
    if err != nil {
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "23505" {
            return nil, connect.NewError(connect.CodeAlreadyExists, fmt.Errorf("<model> already exists"))
        }
        return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("create <model>: %w", err))
    }

    return connect.NewResponse(&<protoAlias>.Create<Model>Response{
        Item: <modelLower>ToProto(row),
    }), nil
}
```

### Get (`rpc_get_<entity>.go`)

```go
func (h *<modelLower>Handler) Get<Model>(
    ctx context.Context,
    req *connect.Request[<protoAlias>.Get<Model>Request],
) (*connect.Response[<protoAlias>.Get<Model>Response], error) {
    id, err := uuid.FromString(req.Msg.GetId())
    if err != nil {
        return nil, connect.NewError(connect.CodeInvalidArgument, fmt.Errorf("invalid id: %w", err))
    }

    row, err := h.store.Get<Model>(ctx, id)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, connect.NewError(connect.CodeNotFound, fmt.Errorf("<model> not found"))
        }
        return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("get <model>: %w", err))
    }

    return connect.NewResponse(&<protoAlias>.Get<Model>Response{
        Item: <modelLower>ToProto(row),
    }), nil
}
```

### List (`rpc_list_<entity>.go`)

Implement cursor-based pagination:
- Default `page_size` to 50 if not set or <= 0
- Decode `page_token` from base64 to get the cursor UUID
- Use `List<Model>sPaginated` query with cursor and page_size
- If no page_token, use `List<Model>s` with LIMIT
- Set `next_page_token` to base64-encoded last UUID if returned rows == page_size

```go
func (h *<modelLower>Handler) List<Model>s(
    ctx context.Context,
    req *connect.Request[<protoAlias>.List<Model>sRequest],
) (*connect.Response[<protoAlias>.List<Model>sResponse], error) {
    pageSize := req.Msg.GetPageSize()
    if pageSize <= 0 {
        pageSize = 50
    }

    var rows []db.<DBType>
    var err error

    if token := req.Msg.GetPageToken(); token != "" {
        cursorBytes, decErr := base64.StdEncoding.DecodeString(token)
        if decErr != nil {
            return nil, connect.NewError(connect.CodeInvalidArgument, fmt.Errorf("invalid page token"))
        }
        cursor, parseErr := uuid.FromString(string(cursorBytes))
        if parseErr != nil {
            return nil, connect.NewError(connect.CodeInvalidArgument, fmt.Errorf("invalid page token"))
        }
        rows, err = h.store.List<Model>sPaginated(ctx, db.List<Model>sPaginatedParams{
            ID:    cursor,
            Limit: pageSize,
        })
    } else {
        rows, err = h.store.List<Model>sPaginated(ctx, db.List<Model>sPaginatedParams{
            ID:    uuid.Must(uuid.FromString("ffffffff-ffff-ffff-ffff-ffffffffffff")),
            Limit: pageSize,
        })
    }
    if err != nil {
        return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("list <model>s: %w", err))
    }

    items := make([]*<protoAlias>.<Model>, len(rows))
    for i, row := range rows {
        items[i] = <modelLower>ToProto(row)
    }

    resp := &<protoAlias>.List<Model>sResponse{Items: items}
    if int32(len(rows)) == pageSize {
        lastID := rows[len(rows)-1].ID.String()
        resp.NextPageToken = base64.StdEncoding.EncodeToString([]byte(lastID))
    }

    return connect.NewResponse(resp), nil
}
```

### Update (`rpc_update_<entity>.go`)

Implement partial updates via FieldMask:
- Fetch current row first
- Apply only masked fields from the request, keep unmasked fields from current row
- FieldMask paths use proto snake_case field names

```go
func (h *<modelLower>Handler) Update<Model>(
    ctx context.Context,
    req *connect.Request[<protoAlias>.Update<Model>Request],
) (*connect.Response[<protoAlias>.Update<Model>Response], error) {
    id, err := uuid.FromString(req.Msg.GetItem().GetEntity().GetId())
    if err != nil {
        return nil, connect.NewError(connect.CodeInvalidArgument, fmt.Errorf("invalid id: %w", err))
    }

    current, err := h.store.Get<Model>(ctx, id)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, connect.NewError(connect.CodeNotFound, fmt.Errorf("<model> not found"))
        }
        return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("get <model>: %w", err))
    }

    // Build update params starting from current values
    params := <modelLower>FromUpdate(<modelLower>ToProto(current), id)

    // Apply masked fields from request
    mask := req.Msg.GetUpdateMask()
    if mask != nil {
        for _, path := range mask.GetPaths() {
            switch path {
            // case "<field_snake>":
            //     params.<FieldPascal> = <mapped value from req.Msg.GetItem()>
            }
        }
    }

    row, err := h.store.Update<Model>(ctx, params)
    if err != nil {
        return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("update <model>: %w", err))
    }

    return connect.NewResponse(&<protoAlias>.Update<Model>Response{
        Item: <modelLower>ToProto(row),
    }), nil
}
```

### Delete (`rpc_delete_<entity>.go`)

Soft delete via `SoftDelete<Model>` query:

```go
func (h *<modelLower>Handler) Delete<Model>(
    ctx context.Context,
    req *connect.Request[<protoAlias>.Delete<Model>Request],
) (*connect.Response[<protoAlias>.Delete<Model>Response], error) {
    id, err := uuid.FromString(req.Msg.GetId())
    if err != nil {
        return nil, connect.NewError(connect.CodeInvalidArgument, fmt.Errorf("invalid id: %w", err))
    }

    rows, err := h.store.SoftDelete<Model>(ctx, id)
    if err != nil {
        return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("delete <model>: %w", err))
    }
    if rows == 0 {
        return nil, connect.NewError(connect.CodeNotFound, fmt.Errorf("<model> not found"))
    }

    return connect.NewResponse(&<protoAlias>.Delete<Model>Response{}), nil
}
```

## Phase 4: Verify

- Confirm all generated files follow the layout:
  ```
  <output_dir>/api/
  ├── handler_<entity>.go
  ├── mapper_<entity>.go
  ├── rpc_create_<entity>.go
  ├── rpc_get_<entity>.go
  ├── rpc_list_<entity>.go
  ├── rpc_update_<entity>.go
  └── rpc_delete_<entity>.go
  ```
- Verify imports resolve correctly against `go.mod` and the sqlc-generated `db/` package
- Verify the handler embeds the correct `Unimplemented*ServiceHandler`
- Verify mapper functions reference the correct sqlc types from `db/models.go`
- Present a summary to the user

## Rules

- Always read the sqlc-generated `db/` package before generating mappers and RPCs to ensure type names match
- Always read the service proto files to ensure RPC signatures match
- Only generate RPC files for operations that have corresponding service RPCs
- If an `api/rpc_*.go` file already exists, do NOT overwrite it (it may contain user customizations)
- Handler and mapper files may be regenerated (they are structural, not customized)
- Use the exact error handling patterns: `CodeAlreadyExists` for duplicate key, `CodeNotFound` for missing rows, `CodeInvalidArgument` for bad input, `CodeInternal` for everything else
- Use `pgconn.PgError` with code `"23505"` for duplicate detection in Create
