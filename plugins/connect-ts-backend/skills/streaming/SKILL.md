---
description: Generate Kafka workers and typed consumer stubs from outbox events. Use when the user wants to stream entity events to Kafka and set up consumer groups.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /streaming - Kafka Workers and Consumer Generation

You are generating Kafka streaming infrastructure from outbox events. Your goal is to read existing outbox event definitions and produce BullMQ workers that publish to Kafka, plus typed consumer group stubs.

## Prerequisites

This skill requires **outbox event definitions** from `/outbox` — `outbox/event_*.ts` files. If these do not exist, inform the user to run `/outbox` first.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/streaming src/acme/inventory/v1/`), use it
   - Otherwise, look for existing `outbox/event_*.ts` files and ask the user which entities to generate streaming for
2. Resolve the package path:
   - Workers: `src/<provider>/<domain>/<version>/workers/`
   - Consumers: `src/<provider>/<domain>/<version>/consumers/`
3. Read the outbox event files to understand available event types
4. Ask the user which consumer groups to generate (e.g. audit, index, webhook, notification, or custom names)
5. Verify `kafkajs` is in `package.json` — if not, inform the user to add it

## Codebase Assessment

Before generating, scan for existing Kafka infrastructure, worker/job processing patterns, and consumer patterns. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing streaming code exists, skip and proceed directly.

## Event Envelope

Generate `workers/envelope.ts` (shared across all entities, generate once, never overwrite):

```typescript
export interface EventEnvelope {
  entityId: string;
  operation: string;
  fieldMask?: string[];
  occurredAt: string;
  payload?: unknown;
}
```

## Workers

For each entity, generate:
- `workers/register_<entity_snake>.ts` — a function that registers all BullMQ workers for that entity, plus a topic constant
- `workers/worker_<operation>_<entity_snake>.ts` — one per mutating operation

Each worker processes BullMQ jobs for the corresponding outbox event. It constructs an `EventEnvelope` and publishes to Kafka using the entity ID as the message key (for partition ordering). Update workers include the `fieldMask` in the envelope.

### Topic naming convention

`<domain>.<entity_snake>.events.<version>` — derived from the proto package (e.g. `inventory.product.events.v1`).

## Consumer Stubs

Generate `consumers/consumer_<subscriber>_<entity_snake>.ts` per subscriber type.

Each consumer stub defines:
- A handler interface: `<Model><Subscriber>Handler` with `handle<Model>Event(envelope: EventEnvelope): Promise<void>`
- A `run<Model><Subscriber>Consumer` function that creates a KafkaJS consumer with the correct topic and group ID, subscribes, and delegates to the handler

### Consumer group ID convention

`<domain>.<entity_snake>.<subscriber_lower>` (e.g. `inventory.product.audit`)

## Verify

- Confirm layout: `workers/envelope.ts`, `workers/register_*.ts`, `workers/worker_*_*.ts`, `consumers/consumer_*_*.ts`
- Verify worker types reference correct outbox event types
- Verify topic names and consumer group IDs follow conventions
- Present a summary to the user

## Rules

- Always read outbox event files before generating to ensure type names match
- Only generate workers for operations that have corresponding outbox events
- The envelope.ts file is shared — generate once, never overwrite
- Consumer stubs define an interface — the user implements the handler
- If worker or consumer files already exist, ask the user before overwriting
- Workers must use the entity ID as the Kafka message key for partition ordering
