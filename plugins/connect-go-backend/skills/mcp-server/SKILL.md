---
description: Generate an MCP server entrypoint under cmd/ that wires handler implementations in-process and registers MCP tools. Use when the user wants to expose their API as an MCP server.
argument-hint: <service name or cmd directory>
disable-model-invocation: true
---

# /mcp-server - MCP Server Entrypoint

You are generating an MCP server entrypoint under `cmd/`. Your goal is to produce a `main.go` that instantiates Connect-RPC handlers in-process, registers MCP tools against them, and runs the MCP server over stdio.

## Prerequisites

This skill requires:
- **Configuration package** from `/config` — `internal/config/` with `Config` struct and `Load()` function
- **Handler constructors** from `/api-handlers` — `api/handler_*.go` files exporting `New<Model>ServiceHandler`
- **MCP tool registries** from `/mcp-tools` — `mcp/registry_*.go` files exporting `Register<Model>Tools`
- **`go.mod`** — for the module path and MCP SDK detection

If prerequisites are missing, inform the user which skills to run first.

## Setup

1. Determine the target name:
   - Default: `cmd/mcp-server/main.go`
   - If the user provides a name (e.g. `/mcp-server inventory`), use `cmd/mcp-<name>/main.go`
   - The `mcp-` prefix is always applied — it identifies MCP tool entrypoints
2. Read `go.mod` for the module path and MCP SDK
3. Scan for existing `api/handler_*.go` files to discover handler constructors
4. Scan for existing `mcp/registry_*.go` files to discover tool registries
5. Check for existing `cmd/` directories to avoid overwriting

## Codebase Assessment

Before generating, scan for an existing MCP server entrypoint or stdio server patterns. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing MCP server code exists, skip and proceed directly.

## Generation

### main.go

Generate `cmd/<service-name>/main.go` with the following structure:

#### Configuration

- Use `config.Load()` from `internal/config/` to resolve all environment variables
- Exit with a clear error if `config.Load()` fails
- Use `pgxpool.New` with `cfg.DatabaseURL` to create the database connection pool

#### In-process handler instantiation

- Create handler instances using their constructors — these are the same handlers used by `/api-server`, but instantiated in-process rather than behind HTTP:

```go
pool, err := pgxpool.New(ctx, cfg.DatabaseURL)
// ...

productHandler := api.NewProductServiceHandler(api.ProductDeps{Pool: pool}, opts...)
categoryHandler := api.NewCategoryServiceHandler(api.CategoryDeps{Pool: pool}, opts...)
```

The handlers return `(string, http.Handler)` — the MCP tool registries accept the Connect service handler interface, so pass the concrete handler directly.

#### MCP server setup

- Create an MCP server instance and register all tool registries:

```go
server := mcpserver.NewServer("service-name", "version")

mcp.RegisterProductTools(server, productHandler)
mcp.RegisterCategoryTools(server, categoryHandler)
```

- Detect which MCP SDK is in `go.mod`:
  - `github.com/modelcontextprotocol/go-sdk` (official, preferred) — use `mcpserver.NewServer` and `mcpserver.ServeStdio`
  - `github.com/mark3labs/mcp-go` (community) — adapt to that SDK's server API
  - If neither is present, inform the user to add the official SDK

#### Stdio transport

- Run the MCP server over stdio — this is the standard transport for Claude integration:

```go
if err := mcpserver.ServeStdio(server); err != nil {
    slog.Error("mcp server error", "error", err)
    os.Exit(1)
}
```

#### Cleanup

- Close the database pool when the server exits
- Use `log/slog` for structured logging, writing to stderr (stdout is reserved for MCP protocol)

```go
slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stderr, nil)))
```

## Verify

- Confirm `cmd/<service-name>/main.go` exists with `package main`
- Verify all discovered handler constructors are instantiated
- Verify all discovered tool registries are wired to the MCP server
- Verify handlers are passed to tool registries as the Connect service handler interface
- Verify slog writes to stderr, not stdout
- Verify database pool is created and closed properly
- Present a summary to the user

## Rules

- Always scan for existing handler constructors and tool registries before generating
- Handlers are instantiated in-process — MCP tools delegate to them directly, not over HTTP
- The MCP server runs over stdio — stdout is reserved for the MCP protocol, all logging goes to stderr
- Do not hardcode handler or tool registrations — discover them from `api/` and `mcp/` packages
- If `cmd/<service-name>/main.go` already exists, ask the user before overwriting
- Use `log/slog` for logging — do not introduce third-party logging libraries
- Keep the entrypoint minimal — configuration, handler instantiation, tool registration, server lifecycle only
