---
description: Generate Connect-RPC service proto definitions from entity messages. Use when the user wants to create service RPCs, request/response types for their proto entities.
argument-hint: <proto file or directory>
disable-model-invocation: true
---

# /service - Connect-RPC Service Proto Generation

You are generating Connect-RPC service `.proto` files from entity message definitions. Your goal is to read proto messages and produce service definitions with RPCs, request/response types, and proper imports.

## Setup

1. Determine the proto source:
   - If the user provides a path (e.g. `/service protos/acme/inventory/v1/`), use it
   - Otherwise, search for `.proto` files and ask the user which entities to generate services for
2. Read the proto files and identify entity messages (same criteria as `/schema`)
3. Determine which CRUD operations to generate for each entity:
   - Ask the user which operations each entity needs: Create, Get, List, Update, Delete
   - Default to all five if the user does not specify
4. Resolve the package path from the proto file's `package` declaration:
   - The proto package segments map directly to the folder path: `acme.inventory.v1` becomes the proto directory `protos/acme/inventory/v1/` (or wherever the source protos live)
   - Place generated service `.proto` files alongside the entity proto files in the same directory
   - Read the `go_package` option from the existing proto files and reuse it exactly for generated service protos
   - If no existing `go_package` exists, follow the convention used by the project's proto generation config (e.g. `buf.gen.yaml`) — the proto Go package path is typically separate from the internal application package path

## Codebase Assessment

Before generating anything, scan the existing codebase to understand what already exists and identify divergences from the target conventions. Present your findings to the user before proceeding.

### What to look for

1. **Existing service definitions**:
   - Search for existing `service` declarations in `.proto` files
   - Check whether services are defined inline in the same file as messages or in separate files
   - Check whether existing RPCs follow the Create/Get/List/Update/Delete naming convention
   - Look for existing request/response message types and their structure

2. **Proto file organisation**:
   - Check whether proto files are split by concern (models, refs, services, RPCs) or monolithic
   - Check whether the `go_package` option follows the `<module>/internal/<provider>/<domain>/<version>;<alias>` convention
   - Identify any buf.yaml, buf.gen.yaml, or protoc configuration

3. **Existing RPC patterns**:
   - Check whether existing services use Connect-RPC, gRPC-Go, or Twirp
   - Look for existing request/response patterns (do they use `item` field? `FieldMask` for updates? cursor-based pagination?)
   - Identify custom RPCs beyond standard CRUD

### What to present to the user

Summarise your findings as a short assessment:
- **Matches**: proto files that already follow the target conventions (separate service files, modular RPC messages, etc.)
- **Divergences**: monolithic proto files, inline service definitions, non-standard RPC patterns, different pagination approaches, etc.
- **Proposed plan**: for each divergence, suggest one of:
  - **Adopt as-is**: the existing pattern works and the skill should follow it (e.g. the project uses offset pagination — keep it)
  - **Incremental refactor**: suggest extracting services into separate files, renaming RPCs to match conventions, or adding missing operations
  - **Generate alongside**: generate new service protos in the target layout alongside existing ones

Ask the user to confirm the plan before proceeding to generation. If no existing service protos exist, skip the assessment and proceed directly.

## Phase 1: Service Definition

For each entity, generate `service_<entity_snake>.proto`:

```protobuf
syntax = "proto3";

package <proto_package>;

import "<path_to_models_proto>";
import "google/protobuf/field_mask.proto";  // if Update operation included

option go_package = "<go_package>";

service <Model>Service {
  rpc Create<Model>(Create<Model>Request) returns (Create<Model>Response);
  rpc Get<Model>(Get<Model>Request) returns (Get<Model>Response);
  rpc List<Model>s(List<Model>sRequest) returns (List<Model>sResponse);
  rpc Update<Model>(Update<Model>Request) returns (Update<Model>Response);
  rpc Delete<Model>(Delete<Model>Request) returns (Delete<Model>Response);
}
```

Only include RPCs for the operations the user selected.

## Phase 2: Request/Response Types

For each operation, generate `rpc_<operation>_<entity_snake>.proto`:

### Create

```protobuf
syntax = "proto3";

package <proto_package>;

import "<path_to_models_proto>";

option go_package = "<go_package>";

message Create<Model>Request {
  <Model> item = 1;
}

message Create<Model>Response {
  <Model> item = 1;
}
```

### Get

```protobuf
syntax = "proto3";

package <proto_package>;

import "<path_to_models_proto>";

option go_package = "<go_package>";

message Get<Model>Request {
  string id = 1;
}

message Get<Model>Response {
  <Model> item = 1;
}
```

### List

```protobuf
syntax = "proto3";

package <proto_package>;

import "<path_to_models_proto>";

option go_package = "<go_package>";

message List<Model>sRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message List<Model>sResponse {
  repeated <Model> items = 1;
  string next_page_token = 2;
}
```

### Update

```protobuf
syntax = "proto3";

package <proto_package>;

import "<path_to_models_proto>";
import "google/protobuf/field_mask.proto";

option go_package = "<go_package>";

message Update<Model>Request {
  string id = 1;
  <Model> item = 2;
  google.protobuf.FieldMask update_mask = 3;
}

message Update<Model>Response {
  <Model> item = 1;
}
```

### Delete

```protobuf
syntax = "proto3";

package <proto_package>;

option go_package = "<go_package>";

message Delete<Model>Request {
  string id = 1;
}

message Delete<Model>Response {}
```

## Phase 3: Verify

- Confirm all generated files follow the naming convention:
  ```
  <proto_dir>/
  ├── service_<entity>.proto
  ├── rpc_create_<entity>.proto
  ├── rpc_get_<entity>.proto
  ├── rpc_list_<entity>.proto
  ├── rpc_update_<entity>.proto
  └── rpc_delete_<entity>.proto
  ```
- Verify import paths are correct relative to the project's proto root
- Verify the `go_package` option matches the existing proto files
- Verify the `package` declaration matches the existing proto files
- Present a summary of generated services and RPCs to the user

## Rules

- Always read existing proto files to extract the correct `package` and `go_package` values
- Do not modify existing proto files (models, refs, etc.)
- If a service proto already exists, ask the user whether to regenerate or skip
- Use UUID validation on `id` fields by convention (the handler layer enforces this)
- Follow the exact file naming: `service_<entity_snake>.proto` and `rpc_<operation>_<entity_snake>.proto`
- Each RPC message type goes in its own file to keep protos modular
