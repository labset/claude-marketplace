---
description: Facilitate requirements discussion and milestone planning for a new piece of work. Use when the user wants to define, refine, or capture requirements and delivery milestones.
argument-hint: <name or existing spec path>
disable-model-invocation: true
---

# /discuss - Requirements Discussion and Milestone Planning

You are facilitating an interactive requirements discussion session. Your goal is to help the user articulate, refine, and capture requirements and delivery milestones for a new piece of work.

## Setup

1. Determine the spec directory for this session:
   - Run `ls -d .agent/specs/*/` to list existing spec directories
   - The naming convention is `.agent/specs/yyyy-mm-dd-xx-<name>/` where `yyyy-mm-dd` is today's date, `xx` is a two-digit sequence number starting at `01`, and `<name>` is a short kebab-case identifier for the spec
   - If directories already exist for today's date, increment the sequence number
   - The `<name>` is derived from the user's arguments (e.g. `/discuss generic crud api` becomes `generic-crud-api`)
   - If the user provides an existing spec path (e.g. `/discuss .agent/specs/2026-04-26-01-generic-crud-api`), use the specified directory and resume from its current state
2. Create the spec directory if it does not exist
3. Read existing `requirements.md` and `milestones.md` if resuming

## Interactive Loop

Guide the user through a structured conversation. Do NOT write files until the user confirms. Follow this flow:

### Phase 1: Problem Space
- Ask the user what they want to build or solve
- Ask clarifying questions to understand scope, constraints, and success criteria
- Summarise what you've understood and confirm with the user

### Phase 2: Requirements Capture
- Propose a structured set of requirements based on the discussion
- Present them to the user for review
- Iterate until the user is satisfied
- Write the agreed requirements to `.agent/specs/yyyy-mm-dd-xx-<name>/requirements.md`

### Phase 3: Milestone Planning
- Propose delivery milestones that break the work into incremental, demonstrable steps
- Each milestone should have: a name, a description, acceptance criteria, and a status (one of: pending, in-progress, done)
- Present them to the user for review
- Iterate until the user is satisfied
- Write the agreed milestones to `.agent/specs/yyyy-mm-dd-xx-<name>/milestones.md`

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

## Rules

- Keep the conversation focused and moving forward
- Ask one question at a time where possible
- Summarise decisions as you go
- Do not write to files until the user explicitly agrees with the content
- When resuming an existing spec, show current status and ask what the user wants to change
