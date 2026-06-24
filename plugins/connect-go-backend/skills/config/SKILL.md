---
description: Generate a centralised configuration package that resolves environment variables with required/optional semantics. Use when the user wants to consolidate app config in one place.
argument-hint: <env vars or config scope>
disable-model-invocation: true
---

# /config - Centralised Configuration Package

You are generating a configuration package under `internal/config/` that consolidates environment variable resolution into a single typed struct.

Before generating, scan for existing code that overlaps with what this skill produces. If existing patterns are found, present divergences and ask the user to confirm a plan before proceeding. If no existing code is found, proceed directly. After generating, verify expected files exist with correct package declarations and imports, then present a summary. If target files already exist, ask the user before overwriting.

## Setup

1. Read `go.mod` for the module path
2. Scan the codebase for existing `os.Getenv` / `os.LookupEnv` calls and `.env` files
3. Classify discovered vars as required or optional based on usage context
4. If the user provides specific variables (e.g. `/config DATABASE_URL PORT KAFKA_BROKERS`), use those directly

## Generation

Generate two files:

### internal/config/env.go

- `required(key string) (string, error)` — returns value or error if unset/empty
- `optional(key, fallback string) string` — returns value or fallback if unset/empty

### internal/config/config.go

- A `Config` struct with typed fields (use `int`, `bool`, `time.Duration`, `[]string` where appropriate)
- A `Load() (*Config, error)` function that:
  - Uses `required` / `optional` helpers for each variable
  - Collects all errors via `errors.Join` before returning — the user sees every missing variable at once
  - Performs type conversions after validation

## Rules

- Standard library only — no third-party config libraries
- After generating, suggest updating `cmd/` entrypoints to use `config.Load()` instead of inline `os.Getenv` calls
