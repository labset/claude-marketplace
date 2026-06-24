---
description: Generate Kafka workers and typed consumer stubs from outbox events. Use when the user wants to stream entity events to Kafka and set up consumer groups.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /streaming - Kafka Workers and Consumer Generation

You are generating Kafka streaming infrastructure from outbox events. Your goal is to read existing outbox event definitions and produce River workers that publish to Kafka, plus typed consumer group stubs.

See `CONVENTIONS.md` for shared conventions (codebase assessment, verify step, overwrite protection).

## Prerequisites

This skill requires **outbox event args** from `/outbox` — `outbox/event_*.go` files. If these do not exist, inform the user to run `/outbox` first.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/streaming internal/acme/inventory/v1/`), use it
   - Otherwise, look for existing `outbox/event_*.go` files and ask the user which entities to generate streaming for
2. Resolve the package path:
   - Workers: `internal/<provider>/<domain>/<version>/workers/`
   - Consumers: `internal/<provider>/<domain>/<version>/consumers/`
3. Read the outbox event files to understand available event types
4. Ask the user which consumer groups to generate (e.g. audit, index, webhook, notification, or custom names)
5. Verify `github.com/segmentio/kafka-go` is in `go.mod` — if not, inform the user to add it

## Event Envelope

Generate `workers/envelope.go` (shared across all entities, generate once, never overwrite):

```go
type EventEnvelope struct {
    EntityID   string          `json:"entity_id"`
    Operation  string          `json:"operation"`
    FieldMask  []string        `json:"field_mask,omitempty"`
    OccurredAt time.Time       `json:"occurred_at"`
    Payload    json.RawMessage `json:"payload,omitempty"`
}
```

## Workers

For each entity, generate:
- `workers/register_<entity_snake>.go` — a `Register<Model>Workers` function that registers all workers for that entity, plus a topic constant
- `workers/worker_<operation>_<entity_snake>.go` — one per mutating operation

Each worker implements `river.Worker` for the corresponding outbox event args. It marshals the event into an `EventEnvelope` and publishes to Kafka using the entity ID as the message key (for partition ordering). Update workers include the `FieldMask` in the envelope.

### Topic naming convention

`<domain>.<entity_snake>.events.<version>` — derived from the proto package (e.g. `inventory.product.events.v1`).

## Consumer Stubs

Generate `consumers/consumer_<subscriber>_<entity_snake>.go` per subscriber type.

Each consumer stub defines:
- A handler interface: `<Model><Subscriber>Handler` with `Handle<Model>Event(ctx, envelope) error`
- A `Run<Model><Subscriber>Consumer` function that creates a `kafka.Reader` with the correct topic and group ID, reads messages in a loop, unmarshals envelopes, and delegates to the handler

### Consumer group ID convention

`<domain>.<entity_snake>.<subscriber_lower>` (e.g. `inventory.product.audit`)

## Rules

- Only generate workers for operations that have corresponding outbox events
- The envelope.go file is shared — generate once, never overwrite
- Consumer stubs define an interface — the user implements the handler
- Workers must use the entity ID as the Kafka message key for partition ordering
