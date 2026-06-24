---
description: Generate a centralised configuration package that resolves environment variables with required/optional semantics. Use when the user wants to consolidate app config in one place.
argument-hint: <env vars or config scope>
disable-model-invocation: true
---

# /config - Centralised Configuration Package

You are generating a configuration package under `internal/config/` that consolidates environment variable resolution into a single struct. Your goal is to provide a typed, validated config that fails fast on missing required variables and applies defaults for optional ones.

## Setup

1. Read `go.mod` for the module path
2. Scan the codebase for existing environment variable usage:
   - `os.Getenv`, `os.LookupEnv` calls in `cmd/`, `internal/`, `main.go`
   - Existing config packages or `.env` files
3. Collect all discovered env vars and classify them as required or optional based on usage context
4. If the user provides specific variables (e.g. `/config DATABASE_URL PORT KAFKA_BROKERS`), use those directly

## Codebase Assessment

Before generating, scan for existing configuration patterns (env parsing libraries, config structs, `.env` loaders). If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing config code exists, skip and proceed directly.

## Generation

### Config struct

Generate `internal/config/config.go` with:

- A `Config` struct with typed fields for each environment variable
- A `Load() (*Config, error)` function that reads from the environment
- Fields use Go types appropriate to the value (e.g. `int` for ports, `time.Duration` for timeouts, `[]string` for comma-separated lists, `string` for everything else)

```go
package config

type Config struct {
    Port        int
    DatabaseURL string
    // ...
}
```

### Environment variable resolution

Generate `internal/config/env.go` with helper functions:

- `required(key string) (string, error)` — returns the value or an error if unset or empty
- `optional(key, fallback string) string` — returns the value or the fallback if unset or empty

The `Load` function uses these helpers and collects all errors before returning, so the user sees every missing variable at once rather than one at a time:

```go
func Load() (*Config, error) {
    var errs []error

    databaseURL, err := required("DATABASE_URL")
    errs = append(errs, err)

    port := optional("PORT", "8080")

    // ... more variables

    if err := errors.Join(errs...); err != nil {
        return nil, fmt.Errorf("config: %w", err)
    }

    portInt, err := strconv.Atoi(port)
    if err != nil {
        return nil, fmt.Errorf("config: PORT: %w", err)
    }

    return &Config{
        DatabaseURL: databaseURL,
        Port:        portInt,
    }, nil
}
```

### Type conversions

Support these conversions based on the struct field type:

| Field type | Conversion |
|---|---|
| `string` | Direct assignment |
| `int` | `strconv.Atoi` |
| `bool` | `strconv.ParseBool` |
| `time.Duration` | `time.ParseDuration` |
| `[]string` | `strings.Split(v, ",")` |

Report conversion errors with the variable name for clarity.

## Verify

- Confirm `internal/config/config.go` and `internal/config/env.go` exist
- Verify all discovered env vars are accounted for in the `Config` struct
- Verify required variables fail with a descriptive error when missing
- Verify optional variables have sensible defaults
- Verify `Load` collects all errors before returning
- Present a summary to the user

## Rules

- Always scan the codebase for existing env var usage before generating
- Use only the standard library — do not introduce third-party config libraries
- Required variables must fail fast with clear error messages naming the variable
- Collect all validation errors before returning so the user sees everything at once
- If `internal/config/` already exists, ask the user before overwriting
- After generating, suggest updating `cmd/` entrypoints to use `config.Load()` instead of inline `os.Getenv` calls
