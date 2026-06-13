---
description: Generate MCP tool wrappers that expose Connect-RPC service operations for Claude integration. Use when the user wants to make their API available as MCP tools.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /mcp - MCP Tool Generation

You are generating MCP (Model Context Protocol) tool wrappers for Connect-RPC services. Your goal is to read existing service handlers and produce MCP tool implementations that expose CRUD operations for Claude integration.

## Prerequisites

This skill requires:
- **Handler implementations** from `/handlers` — `handlers/handler_*.ts`
- **Service proto files** from `/service` — for RPC signatures and request/response types
- **Generated TypeScript** from `buf generate` — Protobuf-ES message types

If these do not exist, inform the user which skills to run first.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/mcp src/acme/inventory/v1/`), use it
   - Otherwise, look for existing `handlers/handler_*.ts` files and ask the user which entities to generate MCP tools for
2. MCP tools go in `src/<provider>/<domain>/<version>/mcp/`
3. Read the service proto files and handler files to understand available operations
4. Verify `@modelcontextprotocol/sdk` is in `package.json` — if not, inform the user to add it

## Codebase Assessment

Before generating, scan for existing MCP implementation, tool patterns, and how tools delegate to backend services. If existing patterns are found, present divergences and a proposed plan. Ask the user to confirm before proceeding. If no existing MCP code exists, skip and proceed directly.

## Tool Registry

Generate `mcp/registry_<entity_snake>.ts` per entity with:
- A factory function that takes the service handler (typed to the Connect-ES service interface)
- Registers one tool per available operation on the MCP server

Tool names use snake_case: `create_<model_snake>`, `get_<model_snake>`, `list_<model_snake>s`, `update_<model_snake>`, `delete_<model_snake>`.

## Tool Implementations

Generate `mcp/tool_<operation>_<entity_snake>.ts` per operation. All tools follow the same pattern:
1. Parse the tool input arguments into the Protobuf-ES request message type
2. Call the handler's RPC method
3. On success: serialize the response as JSON text content
4. On error: return an MCP tool result with `isError: true` and the error message as text

Only generate tools for operations that exist in the service definition.

## Verify

- Confirm layout: `mcp/registry_*.ts`, `mcp/tool_*_*.ts`
- Verify tools reference the Connect-ES service interface
- Verify imports resolve correctly
- Present a summary to the user

## Rules

- Always read service proto and handler files before generating
- MCP tools delegate to the Connect handler interface — they do NOT directly access the database
- Tool errors are returned as MCP results with `isError: true`, not thrown exceptions
- If MCP tool files already exist, ask the user before overwriting
- The registry function takes the service interface, not the concrete handler
