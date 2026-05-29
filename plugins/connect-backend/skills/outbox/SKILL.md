---
description: Add transactional outbox pattern to existing Connect-RPC handlers using River job queue. Use when the user wants reliable event delivery alongside database mutations.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /outbox - Transactional Outbox Pattern

You are adding the transactional outbox pattern to existing Connect-RPC handlers. Your goal is to read the current handler implementations and modify them to wrap mutating operations (create, update, delete) in database transactions with River job queue event insertion.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/outbox internal/acme/inventory/v1/`), use it
   - Otherwise, look for existing `api/handler_*.go` files and ask the user which entities to add outbox to
2. Read the existing handler files in `api/` to understand the current structure
3. Read `go.mod` for module path and verify `github.com/riverqueue/river` is a dependency
   - If River is not in `go.mod`, inform the user they need to add it: `go get github.com/riverqueue/river`
4. Determine the output directory for event args: `outbox/` alongside `api/` and `db/`

## Phase 1: Event Args

Generate `outbox/event_<operation>_<entity_snake>.go` for each mutating operation (create, update, delete):

### Create Event

```go
package outbox

import (
    "time"

    "github.com/gofrs/uuid/v5"
    "github.com/riverqueue/river"
)

type Create<Model>EventArgs struct {
    EntityID   uuid.UUID `json:"entity_id"`
    OccurredAt time.Time `json:"occurred_at"`
}

func (Create<Model>EventArgs) Kind() string {
    return "create_<model_snake>"
}

func (Create<Model>EventArgs) InsertOpts() river.InsertOpts {
    return river.InsertOpts{}
}
```

### Update Event

```go
package outbox

import (
    "time"

    "github.com/gofrs/uuid/v5"
    "github.com/riverqueue/river"
)

type Update<Model>EventArgs struct {
    EntityID   uuid.UUID `json:"entity_id"`
    FieldMask  []string  `json:"field_mask,omitempty"`
    OccurredAt time.Time `json:"occurred_at"`
}

func (Update<Model>EventArgs) Kind() string {
    return "update_<model_snake>"
}

func (Update<Model>EventArgs) InsertOpts() river.InsertOpts {
    return river.InsertOpts{}
}
```

### Delete Event

```go
package outbox

import (
    "time"

    "github.com/gofrs/uuid/v5"
    "github.com/riverqueue/river"
)

type Delete<Model>EventArgs struct {
    EntityID   uuid.UUID `json:"entity_id"`
    OccurredAt time.Time `json:"occurred_at"`
}

func (Delete<Model>EventArgs) Kind() string {
    return "delete_<model_snake>"
}

func (Delete<Model>EventArgs) InsertOpts() river.InsertOpts {
    return river.InsertOpts{}
}
```

## Phase 2: Modify Handler Struct

Update `api/handler_<entity_snake>.go` to include River client and pgxpool:

- Add `pool *pgxpool.Pool` and `river *river.Client[pgx.Tx]` fields to the handler struct
- Update `<Model>Deps` to include `Pool *pgxpool.Pool` and `River *river.Client[pgx.Tx]`
- Update the constructor to wire both fields

```go
type <Model>Deps struct {
    Pool  *pgxpool.Pool
    River *river.Client[pgx.Tx]
}

type <modelLower>Handler struct {
    <connectAlias>.Unimplemented<Model>ServiceHandler
    pool  *pgxpool.Pool
    river *river.Client[pgx.Tx]
    store *db.Queries
}

func New<Model>ServiceHandler(deps <Model>Deps, opts ...connect.HandlerOption) (string, http.Handler) {
    h := &<modelLower>Handler{
        pool:  deps.Pool,
        river: deps.River,
        store: db.New(deps.Pool),
    }
    return <connectAlias>.New<Model>ServiceHandler(h, opts...)
}
```

## Phase 3: Modify Mutating RPCs

Update the create, update, and delete RPC implementations to use transactions and insert outbox events.

### Pattern for all mutating RPCs

Wrap the database operation and event insertion in a transaction:

```go
tx, err := h.pool.Begin(ctx)
if err != nil {
    return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("begin transaction: %w", err))
}
defer tx.Rollback(ctx)

// Use transactional store
txStore := h.store.WithTx(tx)

// ... perform the database operation using txStore instead of h.store ...

// Insert outbox event
_, err = h.river.InsertTx(ctx, tx, outbox.Create<Model>EventArgs{
    EntityID:   id,
    OccurredAt: time.Now(),
}, nil)
if err != nil {
    return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("insert event: %w", err))
}

if err := tx.Commit(ctx); err != nil {
    return nil, connect.NewError(connect.CodeInternal, fmt.Errorf("commit: %w", err))
}
```

For Update operations, also pass the FieldMask paths:

```go
_, err = h.river.InsertTx(ctx, tx, outbox.Update<Model>EventArgs{
    EntityID:   id,
    FieldMask:  req.Msg.GetUpdateMask().GetPaths(),
    OccurredAt: time.Now(),
}, nil)
```

### Non-mutating RPCs

Do NOT modify Get and List RPCs. They do not participate in the outbox pattern.

## Phase 4: Verify

- Confirm the outbox event files exist:
  ```
  <output_dir>/outbox/
  ├── event_create_<entity>.go
  ├── event_update_<entity>.go
  └── event_delete_<entity>.go
  ```
- Verify the handler struct has `pool` and `river` fields
- Verify mutating RPCs use `h.pool.Begin(ctx)`, `h.store.WithTx(tx)`, and `h.river.InsertTx`
- Verify non-mutating RPCs (Get, List) are unchanged
- Verify imports include `github.com/riverqueue/river` and the `outbox` package
- Present a summary of changes to the user

## Rules

- Always read the existing handler and RPC files before modifying them
- Preserve all existing error handling logic (duplicate detection, not found, etc.)
- The transaction wraps BOTH the database operation AND the River event insertion — this is the guarantee of the outbox pattern
- Always `defer tx.Rollback(ctx)` immediately after `Begin`
- Always commit after successful event insertion
- Get and List operations are never modified
- If outbox event files already exist, ask the user before overwriting
