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

For each operation, generate `rpc_<operation>_<entity_snake>.proto`.

All RPC protos must import `buf/validate/validate.proto` when they contain validation annotations. Apply `buf.validate` rules to enforce input constraints at the proto level.

### Validation rules

Apply these validation annotations consistently:

| Field type | Annotation | Purpose |
|---|---|---|
| `string id` | `[(buf.validate.field).string.uuid = true]` | Ensure IDs are valid UUIDs |
| `int32 page_size` | `[(buf.validate.field).int32 = {gte: 0, lte: 100}]` | Bound page size to prevent oversized responses |
| `repeated` fields on requests | `[(buf.validate.field).repeated.max_items = <limit>]` | Prevent unbounded request payloads |
| `string` fields with known bounds | `[(buf.validate.field).string.max_len = <limit>]` | Prevent oversized string inputs |
| `FieldMask update_mask` | `[(buf.validate.field).repeated.max_items = 50]` | Bound the number of fields in an update mask |

For entity model messages: if the model contains `repeated` fields, add `max_items` constraints appropriate to the domain. Ask the user for sensible limits if unclear — do not leave repeated fields unbounded.

### Create

```protobuf
syntax = "proto3";

package <proto_package>;

import "<path_to_models_proto>";
import "buf/validate/validate.proto";

option go_package = "<go_package>";

message Create<Model>Request {
  <Model> item = 1 [(buf.validate.field).required = true];
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
import "buf/validate/validate.proto";

option go_package = "<go_package>";

message Get<Model>Request {
  string id = 1 [(buf.validate.field).string.uuid = true];
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
import "buf/validate/validate.proto";

option go_package = "<go_package>";

message List<Model>sRequest {
  int32 page_size = 1 [(buf.validate.field).int32 = {gte: 0, lte: 100}];
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
import "buf/validate/validate.proto";
import "google/protobuf/field_mask.proto";

option go_package = "<go_package>";

message Update<Model>Request {
  string id = 1 [(buf.validate.field).string.uuid = true];
  <Model> item = 2 [(buf.validate.field).required = true];
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

import "buf/validate/validate.proto";

option go_package = "<go_package>";

message Delete<Model>Request {
  string id = 1 [(buf.validate.field).string.uuid = true];
}

message Delete<Model>Response {}
```

## Phase 3: Entity Model Validation

After generating the RPC protos, review the entity model messages for fields that need validation constraints. For each entity message:

1. **Repeated fields**: add `(buf.validate.field).repeated.max_items` to prevent unbounded collections. Ask the user for appropriate limits based on the domain.
2. **String fields**: consider adding `(buf.validate.field).string.max_len` for fields that should have bounded length (names, descriptions, URLs, etc.)
3. **Enum fields**: add `(buf.validate.field).enum.defined_only = true` to reject unknown enum values
4. **Nested messages with repeated fields**: recursively check nested messages for unbounded collections

Present the proposed validation annotations to the user for confirmation before modifying model proto files.

### Example

For a `Product` message with a `repeated string tags` field:

```protobuf
message Product {
  labset.data.v1.Entity entity = 1;
  string name = 2 [(buf.validate.field).string.max_len = 255];
  string description = 3 [(buf.validate.field).string.max_len = 5000];
  repeated string tags = 4 [(buf.validate.field).repeated.max_items = 20];
  ProductStatus status = 5 [(buf.validate.field).enum.defined_only = true];
}
```

## Phase 4: Verify

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
- Verify all `id` fields have `buf.validate` UUID constraint
- Verify `page_size` fields have bounded range constraints
- Verify `repeated` fields on request messages have `max_items` constraints
- Verify entity model `repeated` fields have been reviewed for bounds
- Present a summary of generated services, RPCs, and validation rules to the user

## Rules

- Always read existing proto files to extract the correct `package` and `go_package` values
- Do not modify existing proto files (models, refs, etc.) without presenting proposed validation annotations to the user first
- If a service proto already exists, ask the user whether to regenerate or skip
- Always use `buf/validate/validate.proto` for input validation — do not rely solely on handler-level checks
- Never leave `repeated` fields on request messages unbounded — always add `max_items`
- Never leave `id` fields without UUID validation
- Follow the exact file naming: `service_<entity_snake>.proto` and `rpc_<operation>_<entity_snake>.proto`
- Each RPC message type goes in its own file to keep protos modular
- If the project does not yet depend on `buf/validate`, inform the user to add it to their `buf.yaml` deps
