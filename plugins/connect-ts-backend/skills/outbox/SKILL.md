---
description: Add transactional outbox pattern to existing Connect-RPC handlers using BullMQ. Use when the user wants reliable event delivery alongside database mutations.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /outbox - Transactional Outbox Pattern

You are adding the transactional outbox pattern to existing Connect-RPC handlers. Your goal is to wrap mutating operations (create, update, delete) in database transactions with BullMQ job insertion for reliable event delivery.

## Prerequisites

This skill requires artifacts from prior skills:
- **Handler implementations** from `/handlers` — `handlers/handler_*.ts` and `handlers/rpc_*.ts`
- **Drizzle queries** from `/schema` — query helpers that accept a transaction argument

If these do not exist, inform the user which skills to run first.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/outbox src/acme/inventory/v1/`), use it
   - Otherwise, look for existing `handlers/handler_*.ts` files and ask the user which entities to add outbox to
2. Resolve the package path — outbox event definitions go in `src/<provider>/<domain>/<version>/outbox/`
3. Read the existing handler files
4. Verify `bullmq` is in `package.json` — if not, inform the user to add it

## Codebase Assessment

Before generating, scan for existing event/messaging patterns, transaction handling, and job queue usage. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing event code exists, skip and proceed directly.

## Event Definitions

Generate `outbox/event_<operation>_<entity_snake>.ts` for each mutating operation (create, update, delete).

Each event definition exports:
- A TypeScript type for the event data: `entityId: string`, `occurredAt: string` (ISO 8601), plus `fieldMask: string[]` (update only)
- A queue name constant: `<operation>_<model_snake>` (e.g. `create_product`)
- A helper function to create the job data from handler arguments

## Handler Modifications

Update the handler deps type to include a `Queue` instance from BullMQ.

## Mutating RPC Modifications

Update create, update, and delete RPC implementations to use the transactional outbox pattern:

1. Start a Drizzle transaction via `db.transaction(async (tx) => { ... })`
2. Perform the database operation using `tx` instead of `db`
3. Insert an outbox record into an `outbox_events` table within the same transaction (entity ID, event type, payload)
4. After the transaction commits, enqueue the BullMQ job from the outbox record

Alternatively, if the project prefers polling-based outbox, generate an `outbox_events` Drizzle table and a poller that reads pending events and enqueues them to BullMQ.

Do NOT modify Get and List RPCs — they do not participate in the outbox pattern.

## Verify

- Confirm event files exist in `outbox/` for each mutating operation
- Verify handler deps include the BullMQ queue
- Verify mutating RPCs use transactions
- Verify non-mutating RPCs are unchanged
- Present a summary to the user

## Rules

- Always read existing handler and RPC files before modifying
- Preserve all existing error handling logic
- The transaction wraps BOTH the database operation AND the outbox record insertion — this is the outbox guarantee
- Get and List operations are never modified
- If outbox event files already exist, ask the user before overwriting
