---
description: Generate MCP tool wrappers that expose Connect-RPC service operations for Claude integration. Use when the user wants to make their API available as MCP tools.
argument-hint: <entity or output directory>
disable-model-invocation: true
---

# /mcp - MCP Tool Generation

You are generating MCP (Model Context Protocol) tool wrappers for Connect-RPC services. Your goal is to read existing service handlers and produce MCP tool implementations that expose CRUD operations for Claude integration.

## Setup

1. Determine the target:
   - If the user provides a path (e.g. `/mcp internal/acme/inventory/v1/`), use it
   - Otherwise, look for existing `api/handler_*.go` files and ask the user which entities to generate MCP tools for
2. Read the service proto files to understand RPC signatures and request/response types
3. Read the handler files to understand available operations
4. Read `go.mod` for module path and verify MCP SDK dependency:
   - Check for `github.com/mark3labs/mcp-go` or the project's preferred MCP SDK
   - If not present, inform the user to add it
5. Output directory: `mcp/` alongside `api/`, `db/`, etc.

## Phase 1: Tool Registry

Generate `mcp/registry_<entity_snake>.go` for each entity:

```go
package mcp

import (
    "github.com/mark3labs/mcp-go/mcp"
    "github.com/mark3labs/mcp-go/server"

    <connectAlias> "<connectImport>"
)

type <modelSnake>Tools struct {
    handler <connectAlias>.<Model>ServiceHandler
}

func Register<Model>Tools(s *server.MCPServer, handler <connectAlias>.<Model>ServiceHandler) {
    t := &<modelSnake>Tools{handler: handler}

    s.AddTool(mcp.NewTool(
        "create_<model_snake>",
        mcp.WithDescription("Create a new <model_lower>"),
        mcp.WithString("item", mcp.Description("JSON representation of the <model_lower> to create"), mcp.Required()),
    ), t.createTool)

    s.AddTool(mcp.NewTool(
        "get_<model_snake>",
        mcp.WithDescription("Get a <model_lower> by ID"),
        mcp.WithString("id", mcp.Description("The UUID of the <model_lower>"), mcp.Required()),
    ), t.getTool)

    s.AddTool(mcp.NewTool(
        "list_<model_snake>s",
        mcp.WithDescription("List <model_lower>s with pagination"),
        mcp.WithNumber("page_size", mcp.Description("Number of items per page")),
        mcp.WithString("page_token", mcp.Description("Pagination token from previous response")),
    ), t.listTool)

    s.AddTool(mcp.NewTool(
        "update_<model_snake>",
        mcp.WithDescription("Update a <model_lower>"),
        mcp.WithString("item", mcp.Description("JSON representation of the <model_lower> to update"), mcp.Required()),
        mcp.WithString("update_mask", mcp.Description("Comma-separated list of fields to update")),
    ), t.updateTool)

    s.AddTool(mcp.NewTool(
        "delete_<model_snake>",
        mcp.WithDescription("Delete a <model_lower> by ID"),
        mcp.WithString("id", mcp.Description("The UUID of the <model_lower>"), mcp.Required()),
    ), t.deleteTool)
}
```

Only register tools for operations that exist in the service definition.

## Phase 2: Tool Implementations

Generate one file per operation: `mcp/tool_<operation>_<entity_snake>.go`

### Create (`tool_create_<entity>.go`)

```go
package mcp

import (
    "context"
    "encoding/json"
    "fmt"

    "connectrpc.com/connect"
    "github.com/mark3labs/mcp-go/mcp"

    <protoAlias> "<protoImport>"
)

func (t *<modelSnake>Tools) createTool(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    itemJSON, _ := req.Params.Arguments["item"].(string)

    var item <protoAlias>.<Model>
    if err := json.Unmarshal([]byte(itemJSON), &item); err != nil {
        return nil, fmt.Errorf("unmarshal item: %w", err)
    }

    resp, err := t.handler.Create<Model>(ctx, connect.NewRequest(&<protoAlias>.Create<Model>Request{
        Item: &item,
    }))
    if err != nil {
        return mcp.NewToolResultError(err.Error()), nil
    }

    data, _ := json.Marshal(resp.Msg)
    return mcp.NewToolResultText(string(data)), nil
}
```

### Get (`tool_get_<entity>.go`)

```go
package mcp

import (
    "context"
    "encoding/json"

    "connectrpc.com/connect"
    "github.com/mark3labs/mcp-go/mcp"

    <protoAlias> "<protoImport>"
)

func (t *<modelSnake>Tools) getTool(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    id, _ := req.Params.Arguments["id"].(string)

    resp, err := t.handler.Get<Model>(ctx, connect.NewRequest(&<protoAlias>.Get<Model>Request{
        Id: id,
    }))
    if err != nil {
        return mcp.NewToolResultError(err.Error()), nil
    }

    data, _ := json.Marshal(resp.Msg)
    return mcp.NewToolResultText(string(data)), nil
}
```

### List (`tool_list_<entity>.go`)

```go
package mcp

import (
    "context"
    "encoding/json"

    "connectrpc.com/connect"
    "github.com/mark3labs/mcp-go/mcp"

    <protoAlias> "<protoImport>"
)

func (t *<modelSnake>Tools) listTool(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    var pageSize int32
    if ps, ok := req.Params.Arguments["page_size"].(float64); ok {
        pageSize = int32(ps)
    }
    pageToken, _ := req.Params.Arguments["page_token"].(string)

    resp, err := t.handler.List<Model>s(ctx, connect.NewRequest(&<protoAlias>.List<Model>sRequest{
        PageSize:  pageSize,
        PageToken: pageToken,
    }))
    if err != nil {
        return mcp.NewToolResultError(err.Error()), nil
    }

    data, _ := json.Marshal(resp.Msg)
    return mcp.NewToolResultText(string(data)), nil
}
```

### Update (`tool_update_<entity>.go`)

```go
package mcp

import (
    "context"
    "encoding/json"
    "fmt"
    "strings"

    "connectrpc.com/connect"
    "github.com/mark3labs/mcp-go/mcp"
    "google.golang.org/protobuf/types/known/fieldmaskpb"

    <protoAlias> "<protoImport>"
)

func (t *<modelSnake>Tools) updateTool(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    itemJSON, _ := req.Params.Arguments["item"].(string)

    var item <protoAlias>.<Model>
    if err := json.Unmarshal([]byte(itemJSON), &item); err != nil {
        return nil, fmt.Errorf("unmarshal item: %w", err)
    }

    updateReq := &<protoAlias>.Update<Model>Request{Item: &item}

    if maskStr, ok := req.Params.Arguments["update_mask"].(string); ok && maskStr != "" {
        updateReq.UpdateMask = &fieldmaskpb.FieldMask{
            Paths: strings.Split(maskStr, ","),
        }
    }

    resp, err := t.handler.Update<Model>(ctx, connect.NewRequest(updateReq))
    if err != nil {
        return mcp.NewToolResultError(err.Error()), nil
    }

    data, _ := json.Marshal(resp.Msg)
    return mcp.NewToolResultText(string(data)), nil
}
```

### Delete (`tool_delete_<entity>.go`)

```go
package mcp

import (
    "context"
    "encoding/json"

    "connectrpc.com/connect"
    "github.com/mark3labs/mcp-go/mcp"

    <protoAlias> "<protoImport>"
)

func (t *<modelSnake>Tools) deleteTool(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    id, _ := req.Params.Arguments["id"].(string)

    resp, err := t.handler.Delete<Model>(ctx, connect.NewRequest(&<protoAlias>.Delete<Model>Request{
        Id: id,
    }))
    if err != nil {
        return mcp.NewToolResultError(err.Error()), nil
    }

    data, _ := json.Marshal(resp.Msg)
    return mcp.NewToolResultText(string(data)), nil
}
```

## Phase 3: Verify

- Confirm all generated files follow the layout:
  ```
  <output_dir>/mcp/
  ├── registry_<entity>.go
  ├── tool_create_<entity>.go
  ├── tool_get_<entity>.go
  ├── tool_list_<entity>.go
  ├── tool_update_<entity>.go
  └── tool_delete_<entity>.go
  ```
- Verify the tools struct references the correct Connect service handler interface
- Verify tool names follow the `<operation>_<model_snake>` convention
- Verify imports resolve correctly
- Present a summary of registered tools to the user

## Rules

- Always read the service proto and handler files before generating MCP tools
- Only generate tool files for operations that exist in the service
- MCP tools delegate to the Connect handler interface — they do NOT directly access the database
- Tool errors are returned as `mcp.NewToolResultError`, not as Go errors (so the MCP client sees them)
- If MCP tool files already exist, ask the user before overwriting
- Tool names use snake_case: `create_product`, `get_product`, `list_products`, etc.
- The registry function takes the Connect service handler interface, not the concrete handler struct
