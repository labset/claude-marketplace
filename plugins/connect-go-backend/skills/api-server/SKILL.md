---
description: Generate a Connect-RPC API server main package under cmd/ with graceful shutdown and handler wiring. Use when the user wants to set up or update their server entrypoint.
argument-hint: <service name or cmd directory>
disable-model-invocation: true
---

# /api-server - Connect-RPC Server Entrypoint

You are generating a Connect-RPC API server entrypoint under `cmd/`. Your goal is to produce a `main.go` that wires handler constructors, starts an HTTP server, and shuts down gracefully on OS signals.

## Prerequisites

This skill requires:
- **Configuration package** from `/config` — `internal/config/` with `Config` struct and `Load()` function
- **Handler constructors** from `/api-handlers` — `api/handler_*.go` files exporting `New<Model>ServiceHandler(deps, opts...) (string, http.Handler)`
- **`go.mod`** — for the module path and to resolve internal imports

If the config package does not exist, inform the user to run `/config` first. If no handler constructors exist, inform the user to run `/api-handlers` first.

## Setup

1. Determine the service name:
   - If the user provides a name (e.g. `/api-server inventory`), use it as the binary name under `cmd/`
   - Otherwise, infer from the module path or ask the user
2. Read `go.mod` for the module path
3. Scan for existing `api/handler_*.go` files to discover available handler constructors and their `Deps` types
4. Check for existing `cmd/` directories to avoid overwriting

## Codebase Assessment

Before generating, scan for an existing server entrypoint, HTTP framework usage, and configuration patterns. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing server code exists, skip and proceed directly.

## Generation

### main.go

Generate `cmd/<service-name>/main.go` with the following structure:

#### Configuration

- Use `config.Load()` from `internal/config/` to resolve all environment variables
- Access typed fields from the returned `*config.Config` (e.g. `cfg.Port`, `cfg.DatabaseURL`)
- Exit with a clear error if `config.Load()` fails — all missing variables are reported at once
- Use `pgxpool.New` with `cfg.DatabaseURL` to create the database connection pool

#### Handler wiring

- Import each `api/` package that exports handler constructors
- Create a `*http.ServeMux` and register each handler:

```go
mux := http.NewServeMux()

// wire each handler constructor
path, handler := api.New<Model>ServiceHandler(deps, opts...)
mux.Handle(path, handler)
```

- If `connectrpc.com/validate` is in `go.mod`, add `validate.NewInterceptor()` to the shared handler options
- If CORS support is needed, wrap the mux with `connectrpc.com/cors` or `rs/cors`

#### HTTP server

- Create an `http.Server` with the mux and configured address
- Support both HTTP/2 and HTTP/1.1 via `golang.org/x/net/http2` and `golang.org/x/net/http2/h2c`:

```go
server := &http.Server{
    Addr:    fmt.Sprintf(":%d", cfg.Port),
    Handler: h2c.NewHandler(mux, &http2.Server{}),
}
```

#### Graceful shutdown

- Listen for `SIGINT` and `SIGTERM` using `os/signal`
- On signal, call `server.Shutdown(ctx)` with a timeout context
- Close the database pool after the server stops
- Use `log/slog` for structured logging of lifecycle events

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() {
    slog.Info("server started", "addr", server.Addr)
    if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        slog.Error("server error", "error", err)
        os.Exit(1)
    }
}()

<-ctx.Done()
slog.Info("shutting down")

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := server.Shutdown(shutdownCtx); err != nil {
    slog.Error("shutdown error", "error", err)
}
pool.Close()
```

## Verify

- Confirm `cmd/<service-name>/main.go` exists with `package main`
- Verify all discovered handler constructors are wired into the mux
- Verify graceful shutdown handles both `SIGINT` and `SIGTERM`
- Verify database pool is created and closed properly
- Verify h2c is configured for HTTP/2 support
- Present a summary to the user

## Rules

- Always scan for existing handler constructors before generating to ensure all services are wired
- Do not hardcode handler registrations — discover them from the `api/` package
- If `cmd/<service-name>/main.go` already exists, ask the user before overwriting
- Use `log/slog` for logging — do not introduce third-party logging libraries
- Do not add middleware or interceptors beyond what is discovered in `go.mod` (e.g. validate)
- Keep the entrypoint minimal — configuration, wiring, server lifecycle only
