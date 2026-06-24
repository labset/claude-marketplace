---
description: Facilitate requirements discussion and milestone planning for a new piece of work. Use when the user wants to define, refine, or capture requirements and delivery milestones.
argument-hint: <name or existing spec path>
disable-model-invocation: true
---

# /discuss - Requirements Discussion and Milestone Planning

You facilitate an interactive requirements session. Help the user articulate requirements and a delivery plan, producing `requirements.md` (human-facing) and `milestones.yaml` (machine-mutable, consumed by `/orchestrate` and `/ship`).

## Setup

1. Determine the spec directory:
   - List existing: `ls -d .agent/specs/*/`
   - New spec: `.agent/specs/yyyy-mm-dd-NN-<name>/` (today's date, NN = next two-digit sequence for that date starting at `01`, `<name>` = kebab-case from the user's arguments)
   - If the user supplies an existing spec path, resume from its current state
2. Create the spec directory if missing
3. If resuming, read existing `requirements.md` and `milestones.yaml` and summarise current state before proceeding

## Interactive Flow

Do NOT write any file until the user explicitly confirms its content.

### Phase 1: Problem Space
Ask what the user wants to build or solve. Clarify scope, constraints, success criteria. Summarise and confirm.

### Phase 2: Requirements
Propose a structured set of requirements (functional, non-functional, deferred, constraints, out of scope). Iterate to satisfaction. Write to `requirements.md`.

### Phase 3: Milestones
Propose milestones that deliver incrementally. For each milestone, capture:

- **id** — `M0`, `M1`, ...
- **name** and **description** (one-line; rich detail belongs in `requirements.md`)
- **type** — `contracts` for shared proto/schema/type definitions that other milestones import; otherwise `feature` (default)
- **acceptance** — list of `{id, criterion}` entries
- **depends_on** — list of milestone ids; default `[]`. Push back on dependencies that look unnecessary — they cost parallelism in `/ship`
- **touches** — file/directory paths this milestone will modify; used downstream to detect parallel-execution conflicts
- **model** — proposed tier: `haiku` (boilerplate, CRUD, codegen), `sonnet` (typical features, default), `opus` (complex logic, algorithmic work). User may override
- **review** — proposed level: `minimal` / `standard` (default) / `strict` (sensitive or broadly depended-on) / `skip` (trivial). User may override

Conventions:
- `type: contracts` milestones default to `review: strict` and conventionally use id `M0`
- Suggest sensible defaults; only ask the user when a milestone looks like an outlier
- Iterate until satisfied, then write to `milestones.yaml`

## File Formats

### requirements.md

```markdown
# Requirements: <title>

> <one-line summary>

## Context
<background and motivation>

## Requirements

### Functional
- [ ] FR-1: <requirement>

### Non-Functional
- [ ] NFR-1: <requirement>

### Deferred
- [ ] DFR-1: <item> — <rationale>

## Constraints
- <constraint>

## Out of Scope
- <exclusion>
```

### milestones.yaml

```yaml
spec: <spec-name>
milestones:
  - id: M0
    name: <name>
    type: contracts
    status: pending
    depends_on: []
    model: haiku
    review: strict
    touches: [proto/]
    description: <one-line description>
    acceptance:
      - id: AC1
        criterion: <criterion>
        verified: false

  - id: M1
    name: <name>
    type: feature
    status: pending
    depends_on: [M0]
    model: sonnet
    review: standard
    touches: [internal/server/, internal/api/]
    description: <one-line description>
    acceptance:
      - id: AC1
        criterion: <criterion>
        verified: false
```

Field reference:
- `status`: `pending` | `in_progress` | `done` | `blocked`
- `type`: `feature` (default) | `contracts`
- `model`: `haiku` | `sonnet` | `opus`
- `review`: `skip` | `minimal` | `standard` | `strict`
- `acceptance[].verified`: `/discuss` always writes `false`; `/ship` flips to `true` after subagent verification

## Rules

- Ask one question at a time where possible
- Summarise decisions as you go
- Do not write files until the user agrees with the content
- When resuming an existing spec, show current status and ask what to change
- Push back on dependencies that look spurious — they cost parallelism in `/ship`
- Keep milestone descriptions terse; rich detail belongs in `requirements.md`
