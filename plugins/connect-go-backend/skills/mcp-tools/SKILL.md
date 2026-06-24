---
description: Generate MCP tool wrappers that expose Connect-RPC service operations for Claude integration. Use when the user wants to make their API available as MCP tools.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /mcp-tools - MCP Tool Generation

You are generating MCP (Model Context Protocol) tool wrappers for Connect-RPC services. Your goal is to read existing service handlers and produce MCP tool implementations that expose CRUD operations for Claude integration.

## Prerequisites

This skill requires:
- **Handler implementations** from `/api-handlers` — `api/handler_*.go`
- **Service proto files** from `/protos` — for RPC signatures and request/response types

If these do not exist, inform the user which skills to run first.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/mcp-tools internal/acme/inventory/v1/`), use it
   - Otherwise, look for existing `api/handler_*.go` files and ask the user which entities to generate MCP tools for
2. MCP tools go in `internal/<provider>/<domain>/<version>/mcp/`
3. Read the service proto files and handler files to understand available operations
4. Detect which MCP SDK is in `go.mod`:
   - `github.com/modelcontextprotocol/go-sdk` (official, preferred)
   - `github.com/mark3labs/mcp-go` (community, adapt if already in use)
   - If neither is present, inform the user to add the official SDK

## Codebase Assessment

Before generating, scan for existing MCP implementation, tool patterns, and how tools delegate to backend services. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing MCP code exists, skip and proceed directly.

## Tool Registry

Generate `mcp/registry_<entity_snake>.go` per entity with:
- A tools struct holding the Connect service handler interface (not the concrete handler)
- A `Register<Model>Tools` function that registers one tool per available operation on the MCP server

Tool names use snake_case: `create_<model_snake>`, `get_<model_snake>`, `list_<model_snake>s`, `update_<model_snake>`, `delete_<model_snake>`.

## Tool Implementations

Generate `mcp/tool_<operation>_<entity_snake>.go` per operation. All tools follow the same pattern:
1. Unmarshal `req.Params.Arguments` directly into the proto request type (do NOT extract individual fields)
2. Call the handler's RPC method via `connect.NewRequest`
3. On success: marshal the response as JSON into a `TextContent` result
4. On error: return a `CallToolResult` with `IsError: true` and the error message as text

Only generate tools for operations that exist in the service definition.

## Verify

- Confirm layout: `mcp/registry_*.go`, `mcp/tool_*_*.go`
- Verify tools struct references the Connect service handler interface
- Verify imports resolve correctly
- Present a summary to the user

## Rules

- Always read service proto and handler files before generating
- MCP tools delegate to the Connect handler interface — they do NOT directly access the database
- Tool errors are returned as `CallToolResult` with `IsError: true`, not as Go errors
- If MCP tool files already exist, ask the user before overwriting
- The registry function takes the Connect service handler interface, not the concrete struct
- If the project uses `mark3labs/mcp-go`, adapt the generated code to match that SDK's API
