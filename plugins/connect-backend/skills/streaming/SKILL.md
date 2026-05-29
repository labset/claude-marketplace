---
description: Generate Kafka workers and typed consumer stubs from outbox events. Use when the user wants to stream entity events to Kafka and set up consumer groups.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /streaming - Kafka Workers and Consumer Generation

You are generating Kafka streaming infrastructure from outbox events. Your goal is to read existing outbox event definitions and produce River workers that publish to Kafka, plus typed consumer group stubs.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/streaming internal/acme/inventory/v1/`), use it
   - Otherwise, look for existing `outbox/event_*.go` files and ask the user which entities to generate streaming for
2. Read the outbox event files to understand available event types
3. Read the proto files or handler files to determine the domain and version for topic naming
4. Determine subscriber types:
   - Ask the user which consumer groups to generate: audit, index, webhook, notification
   - Or accept custom subscriber names
5. Verify `github.com/segmentio/kafka-go` is in `go.mod`
   - If not, inform the user: `go get github.com/segmentio/kafka-go`
6. Output directories: `workers/` and `consumers/` alongside `api/`, `db/`, and `outbox/`

## Phase 1: Event Envelope

Generate `workers/envelope.go` (shared across all entities):

```go
package workers

import (
    "encoding/json"
    "time"
)

type EventEnvelope struct {
    EntityID   string          `json:"entity_id"`
    Operation  string          `json:"operation"`
    FieldMask  []string        `json:"field_mask,omitempty"`
    OccurredAt time.Time       `json:"occurred_at"`
    Payload    json.RawMessage `json:"payload,omitempty"`
}
```

Only generate this file once. If it already exists, skip it.

## Phase 2: Worker Registration

Generate `workers/register_<entity_snake>.go` for each entity:

```go
package workers

import (
    "github.com/riverqueue/river"
    "github.com/segmentio/kafka-go"
)

const <modelLower>Topic = "<domain>.<model_snake>.events.<version>"

type <Model>WorkerDeps struct {
    Writer *kafka.Writer
}

func Register<Model>Workers(workers *river.Workers, deps <Model>WorkerDeps) {
    river.AddWorker(workers, &create<Model>Worker{writer: deps.Writer})
    river.AddWorker(workers, &update<Model>Worker{writer: deps.Writer})
    river.AddWorker(workers, &delete<Model>Worker{writer: deps.Writer})
}
```

### Topic naming convention

- Format: `<domain>.<entity_snake>.events.<version>`
- Derived from proto package: `acme.inventory.v1` with entity `Product` produces `inventory.product.events.v1`
- The domain is the second segment of the proto package

## Phase 3: Worker Implementations

Generate `workers/worker_<operation>_<entity_snake>.go` for each mutating operation:

```go
package workers

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/riverqueue/river"
    "github.com/segmentio/kafka-go"

    "<module>/outbox"
)

type <operation><Model>Worker struct {
    river.WorkerDefaults[outbox.<Operation><Model>EventArgs]
    writer *kafka.Writer
}

func (w *<operation><Model>Worker) Work(ctx context.Context, job *river.Job[outbox.<Operation><Model>EventArgs]) error {
    envelope := EventEnvelope{
        EntityID:   job.Args.EntityID.String(),
        Operation:  "<operation>",
        OccurredAt: job.Args.OccurredAt,
    }
    data, err := json.Marshal(envelope)
    if err != nil {
        return fmt.Errorf("marshal envelope: %w", err)
    }
    return w.writer.WriteMessages(ctx, kafka.Message{
        Topic: <modelLower>Topic,
        Key:   []byte(envelope.EntityID),
        Value: data,
    })
}
```

For the update worker, also include the FieldMask in the envelope:

```go
envelope := EventEnvelope{
    EntityID:   job.Args.EntityID.String(),
    Operation:  "update",
    FieldMask:  job.Args.FieldMask,
    OccurredAt: job.Args.OccurredAt,
}
```

## Phase 4: Consumer Stubs

Generate `consumers/consumer_<subscriber>_<entity_snake>.go` for each subscriber type:

```go
package consumers

import (
    "context"
    "encoding/json"
    "fmt"
    "log/slog"

    "github.com/segmentio/kafka-go"

    "<module>/workers"
)

type <Model><Subscriber>Handler interface {
    Handle<Model>Event(ctx context.Context, envelope workers.EventEnvelope) error
}

func Run<Model><Subscriber>Consumer(ctx context.Context, brokers []string, handler <Model><Subscriber>Handler) error {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: brokers,
        Topic:   "<domain>.<model_snake>.events.<version>",
        GroupID: "<domain>.<model_snake>.<subscriber_lower>",
    })
    defer reader.Close()

    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            if ctx.Err() != nil {
                return nil
            }
            return fmt.Errorf("read message: %w", err)
        }

        var envelope workers.EventEnvelope
        if err := json.Unmarshal(msg.Value, &envelope); err != nil {
            slog.ErrorContext(ctx, "unmarshal envelope", "error", err)
            continue
        }

        if err := handler.Handle<Model>Event(ctx, envelope); err != nil {
            slog.ErrorContext(ctx, "handle event", "error", err, "entity_id", envelope.EntityID)
        }
    }
}
```

### Consumer group ID convention

- Format: `<domain>.<entity_snake>.<subscriber_lower>`
- Example: `inventory.product.audit`

## Phase 5: Verify

- Confirm all generated files follow the layout:
  ```
  <output_dir>/
  ├── workers/
  │   ├── envelope.go
  │   ├── register_<entity>.go
  │   ├── worker_create_<entity>.go
  │   ├── worker_update_<entity>.go
  │   └── worker_delete_<entity>.go
  └── consumers/
      ├── consumer_audit_<entity>.go
      ├── consumer_index_<entity>.go
      └── ...
  ```
- Verify worker types reference the correct outbox event args
- Verify topic names follow the `<domain>.<entity>.events.<version>` convention
- Verify consumer group IDs follow the `<domain>.<entity>.<subscriber>` convention
- Present a summary of workers and consumers generated to the user

## Rules

- Always read outbox event files before generating workers to ensure type names match
- Only generate workers for operations that have corresponding outbox events
- The envelope.go file is shared — generate it once, never overwrite
- Consumer stubs define an interface — the user implements the handler
- If worker or consumer files already exist, ask the user before overwriting
- Use `slog` for logging in consumers, not `log` or `fmt.Println`
- Workers must use the entity ID as the Kafka message key for partition ordering
