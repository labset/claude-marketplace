---
description: Ingest requirements from an external source and produce a spec ready for /ship. Use when requirements already exist and the interactive /discuss phase should be skipped.
argument-hint: <name> --from <path-or-stdin>
disable-model-invocation: true
---

# /ingest - Ingest Requirements into a Spec

You are a spec ingestion agent. Your goal is to take requirements from an external source (a document, issue, or structured input) and produce a well-formed spec that `/ship` can deliver, without interactive discussion.

## Setup

1. Determine the spec directory:
   - Run `ls -d .agent/specs/*/` to list existing spec directories (ignore errors if none exist)
   - The naming convention is `.agent/specs/yyyy-mm-dd-xx-<name>/` where `yyyy-mm-dd` is today's date, `xx` is a two-digit sequence number starting at `01`, and `<name>` is derived from the user's arguments
   - If directories already exist for today's date, increment the sequence number
2. Read the source material:
   - If the user provides a file path (e.g. `/ingest auth-api --from docs/auth-rfc.md`), read that file
   - If no `--from` is specified, ask the user to provide the requirements inline
3. Create the spec directory

## Ingestion Process

### Phase 1: Extract Requirements

Parse the source material and extract:
- Functional requirements — what the system must do
- Non-functional requirements — quality attributes, constraints
- Items that are explicitly deferred or out of scope

Structure them into the standard `requirements.md` format (see below).

### Phase 2: Derive Milestones

Break the requirements into incremental, deliverable milestones:
- Each milestone should be independently verifiable
- Order milestones so that earlier ones unblock later ones
- Each milestone gets acceptance criteria derived from the requirements it covers
- All milestones start with status `pending`

Structure them into the standard `milestones.md` format (see below).

### Phase 3: Write and Report

1. Write `requirements.md` and `milestones.md` to the spec directory
2. Write `.agent/specs/yyyy-mm-dd-xx-<name>/status.json` with the initial status (see format below)
3. Display a summary of what was ingested: requirement count, milestone count, spec path

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
- [ ] FR-2: <requirement>

### Non-Functional

- [ ] NFR-1: <requirement>
- [ ] NFR-2: <requirement>

### Deferred

- [ ] DFR-1: <item explicitly deferred for future work, with brief rationale>

## Constraints

- <constraint>

## Out of Scope

- <exclusion>
```

### milestones.md

```markdown
# Milestones: <title>

## M1: <milestone name>

- **Status:** pending
- **Description:** <what this milestone delivers>
- **Acceptance Criteria:**
  - [ ] <criterion>
  - [ ] <criterion>

## M2: <milestone name>

- **Status:** pending
- **Description:** <what this milestone delivers>
- **Acceptance Criteria:**
  - [ ] <criterion>
  - [ ] <criterion>
```

### status.json

```json
{
  "spec": "<name>",
  "path": ".agent/specs/yyyy-mm-dd-xx-<name>",
  "state": "ready",
  "milestones": {
    "total": 3,
    "done": 0,
    "in_progress": 0,
    "pending": 3
  },
  "blocked": null,
  "updated_at": "yyyy-mm-ddTHH:MM:SSZ"
}
```

## Rules

- Do not ask clarifying questions — make reasonable assumptions and document them in the Context section of requirements.md
- If the source material is ambiguous, note the ambiguity as a constraint and pick the simpler interpretation
- Always write status.json alongside the spec files
- The spec must be immediately actionable by `/ship` with no further human input
