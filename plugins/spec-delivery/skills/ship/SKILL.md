---
description: Build and deliver work defined in a spec, milestone by milestone, ensuring production readiness at each step. Use when the user wants to implement a spec.
argument-hint: <spec path>
disable-model-invocation: true
---

# /ship - Build, Iterate, and Ensure Production Readiness

You are facilitating an interactive delivery session. Your goal is to help the user implement the work defined in a spec, milestone by milestone, ensuring production readiness at each step.

## Setup

1. Determine which spec to deliver:
   - If the user provides arguments (e.g. `/ship specs/2026-04-26-01-generic-crud-api`), use the specified directory
   - Otherwise, run `ls -d specs/*/` to list existing spec directories and ask the user which one to work on
2. Read `specs/yyyy-mm-dd-xx-<name>/requirements.md` and `specs/yyyy-mm-dd-xx-<name>/milestones.md`
3. Display a summary of the current state: which milestones are done, in-progress, or pending

## Interactive Loop

Work through milestones sequentially. For each milestone:

### Phase 1: Plan
- Review the milestone's description and acceptance criteria
- Propose an implementation plan: what files to create/modify, what approach to take
- Confirm the plan with the user before proceeding

### Phase 2: Build
- Implement the changes according to the plan
- Follow all project conventions from CLAUDE.md
- After implementation, run relevant build/test commands to verify:
  - For Go code: use the appropriate pnpm build targets
  - For TypeScript code: use the appropriate pnpm build targets
  - For proto schemas: use buf to regenerate and verify
- Fix any issues that arise

### Phase 3: Verify
- Walk through each acceptance criterion with the user
- For each criterion, confirm it is met or identify what's missing
- Iterate until all criteria are satisfied

### Phase 4: Update Status
- Update `milestones.md` to mark the milestone as done:
  - Set **Status:** to `done`
  - Check off completed acceptance criteria
- Ask the user if they want to proceed to the next milestone

## Production Readiness Checks

Before marking a milestone as done, verify:
- Code compiles without errors
- Tests pass (if applicable)
- No unintended side effects on existing functionality
- Generated code (if any) is up to date

## Rules

- Always read the latest state of requirements.md and milestones.md before starting work
- Work on one milestone at a time, in order
- Do not skip ahead to future milestones without completing current ones
- Keep the user informed of progress throughout
- If a milestone's scope needs to change, suggest updating the spec files first via `/discuss`
- When updating milestones.md, preserve the existing format and only change status and checkboxes
