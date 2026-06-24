# Shared Conventions

These conventions apply to all connect-go-backend skills. Each skill inherits these rules implicitly.

## Codebase Assessment

Before generating, scan for existing code that overlaps with what the skill produces. If existing patterns are found, present divergences and a proposed plan (adopt existing patterns, suggest incremental refactors, or generate alongside). Ask the user to confirm before proceeding. If no existing code is found, skip and proceed directly.

## Overwrite Protection

- If target files already exist, ask the user before overwriting
- Files marked as structural (handlers, mappers, registries) may be regenerated
- Files marked as customisable (RPC implementations, consumer handlers) must NOT be overwritten

## Verify Step

Every skill ends with a verification step:
1. Confirm expected files exist with correct names and package declarations
2. Verify imports resolve against `go.mod`
3. Present a summary of generated files to the user

## Common Rules

- Always read relevant source files before generating — do not assume types or names
- Use only the standard library unless the skill explicitly specifies a dependency
- Use `log/slog` for logging — do not introduce third-party logging libraries
- Use snake_case for file names, PascalCase for Go types, snake_case for SQL and proto file names
- Derive paths from the proto `package` declaration: `<provider>.<domain>.<version>` maps to `internal/<provider>/<domain>/<version>/`

## cmd/ Naming Convention

- `api-` prefix for Connect-RPC API entrypoints (e.g. `cmd/api-server/`)
- `mcp-` prefix for MCP tool entrypoints (e.g. `cmd/mcp-server/`)
- Default names are `api-server` and `mcp-server`; the user can override by passing a name, which gets the prefix applied (e.g. `/server api inventory` produces `cmd/api-inventory/`)
