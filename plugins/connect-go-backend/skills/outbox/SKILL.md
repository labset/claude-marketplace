---
description: Add transactional outbox pattern to existing Connect-RPC handlers using River job queue. Use when the user wants reliable event delivery alongside database mutations.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /outbox - Transactional Outbox Pattern

You are adding the transactional outbox pattern to existing Connect-RPC handlers. Your goal is to wrap mutating operations (create, update, delete) in database transactions with River job queue event insertion.

## Prerequisites

This skill requires artifacts from prior skills:
- **Handler implementations** from `/api-handlers` — `api/handler_*.go` and `api/rpc_*.go`
- **sqlc store** from `/db-schema` + `sqlc generate` — the `db/` package with `Queries.WithTx()`

If these do not exist, inform the user which skills to run first.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/outbox internal/acme/inventory/v1/`), use it
   - Otherwise, look for existing `api/handler_*.go` files and ask the user which entities to add outbox to
2. Resolve the package path — outbox event args go in `internal/<provider>/<domain>/<version>/outbox/`
3. Read the existing handler files in `api/`
4. Verify `github.com/riverqueue/river` is in `go.mod` — if not, inform the user to add it

## Codebase Assessment

Before generating, scan for existing event/messaging patterns, transaction handling, and job queue usage. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing event code exists, skip and proceed directly.

## Event Args

Generate `outbox/event_<operation>_<entity_snake>.go` for each mutating operation (create, update, delete).

Each event arg struct implements River's `JobArgs` interface:
- Fields: `EntityID uuid.UUID`, `OccurredAt time.Time` (all operations), plus `FieldMask []string` (update only)
- `Kind()` returns `"<operation>_<model_snake>"` (e.g. `"create_product"`)
- `InsertOpts()` returns `river.InsertOpts{}`

## Handler Modifications

Update the handler struct in `api/handler_<entity_snake>.go`:
- Add `pool *pgxpool.Pool` and `river *river.Client[pgx.Tx]` fields
- Update `Deps` struct to include `Pool` and `River`
- Update the constructor to wire both fields

## Mutating RPC Modifications

Update create, update, and delete RPC implementations to use the transactional outbox pattern:

1. `h.pool.Begin(ctx)` to start a transaction
2. `defer tx.Rollback(ctx)` immediately after
3. `h.store.WithTx(tx)` for a transactional store
4. Perform the database operation using the transactional store
5. `h.river.InsertTx(ctx, tx, <EventArgs>{...}, nil)` to insert the outbox event
6. `tx.Commit(ctx)` to commit both atomically

Do NOT modify Get and List RPCs — they do not participate in the outbox pattern.

## Verify

- Confirm event files exist in `outbox/` for each mutating operation
- Verify handler struct has `pool` and `river` fields
- Verify mutating RPCs use transactions with `InsertTx`
- Verify non-mutating RPCs are unchanged
- Present a summary to the user

## Rules

- Always read existing handler and RPC files before modifying
- Preserve all existing error handling logic
- The transaction wraps BOTH the database operation AND the River event insertion — this is the outbox guarantee
- Get and List operations are never modified
- If outbox event files already exist, ask the user before overwriting
