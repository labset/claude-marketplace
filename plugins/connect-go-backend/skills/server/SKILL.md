---
description: Generate a server entrypoint under cmd/ for Connect-RPC APIs or MCP tools. Use when the user wants to set up their api or mcp server.
argument-hint: <api|mcp> [name]
disable-model-invocation: true
---

# /server - Server Entrypoint Generation

You are generating a server entrypoint under `cmd/`. The first argument selects the transport:

- `/server api` — HTTP server exposing Connect-RPC handlers (default: `cmd/api-server/`)
- `/server mcp` — stdio server exposing MCP tools backed by in-process handlers (default: `cmd/mcp-server/`)

An optional second argument overrides the name: `/server api inventory` produces `cmd/api-inventory/`.

See `CONVENTIONS.md` for shared conventions (codebase assessment, verify step, overwrite protection, cmd/ naming).

## Prerequisites

Both transports require:
- **Configuration package** from `/config` — `internal/config/` with `Load()` function
- **Handler constructors** from `/api-handlers` — `api/handler_*.go`

The `mcp` transport additionally requires:
- **MCP tool registries** from `/mcp-tools` — `mcp/registry_*.go`

If prerequisites are missing, inform the user which skills to run first.

## Setup

1. Parse the transport argument (`api` or `mcp`) — if omitted, ask the user
2. Apply the naming convention from `CONVENTIONS.md` (`api-` or `mcp-` prefix)
3. Read `go.mod` for the module path
4. Scan for `api/handler_*.go` to discover handler constructors
5. For `mcp` transport, also scan for `mcp/registry_*.go` to discover tool registries

## Generation — shared structure

Both transports share this `main.go` skeleton:

1. **Config** — `config.Load()`, exit on failure
2. **Database pool** — `pgxpool.New(ctx, cfg.DatabaseURL)`, defer close
3. **Handler instantiation** — create handler instances from discovered constructors
4. **Slog setup** — `log/slog` for structured logging

## Generation — api transport

Wire handlers into an HTTP server:

- Register each `(path, handler)` pair on `*http.ServeMux`
- If `connectrpc.com/validate` is in `go.mod`, add the interceptor to shared handler options
- Wrap mux with `h2c.NewHandler` for HTTP/2 cleartext support
- Use `fmt.Sprintf(":%d", cfg.Port)` for the listen address
- Graceful shutdown: `signal.NotifyContext` for `SIGINT`/`SIGTERM`, `server.Shutdown` with a 10s timeout, then close pool

## Generation — mcp transport

Wire handlers into an MCP server over stdio:

- Instantiate handlers in-process (same constructors as `api`, but not behind HTTP)
- Create an MCP server and call each `Register<Model>Tools(server, handler)`
- Detect MCP SDK in `go.mod`: prefer `github.com/modelcontextprotocol/go-sdk`, adapt for `github.com/mark3labs/mcp-go`
- Serve over stdio via `mcpserver.ServeStdio`
- **stdout is reserved for MCP protocol** — set slog default handler to write to stderr
- Close pool on exit
