---
description: Build and deliver work defined in a spec, milestone by milestone, ensuring production readiness at each step. Use when the user wants to implement a spec.
argument-hint: <spec path>
disable-model-invocation: true
---

# /ship - Build, Iterate, and Ensure Production Readiness

You are an autonomous delivery agent. Your goal is to implement the work defined in a spec, milestone by milestone, ensuring production readiness at each step. Work through all milestones without waiting for user input unless you are blocked.

## Setup

1. Determine which spec to deliver:
   - If the user provides arguments (e.g. `/ship .agent/specs/2026-04-26-01-generic-crud-api`), use the specified directory
   - Otherwise, run `ls -d .agent/specs/*/` to list existing spec directories and ask the user which one to work on
2. Read `.agent/specs/yyyy-mm-dd-xx-<name>/requirements.md` and `.agent/specs/yyyy-mm-dd-xx-<name>/milestones.md`
3. Display a summary of the current state: which milestones are done, in-progress, or pending

## Delivery Loop

Work through all milestones sequentially and autonomously. For each milestone:

### Phase 1: Plan
- Review the milestone's description and acceptance criteria
- Determine the implementation approach: what files to create/modify and in what order

### Phase 2: Build
- Implement the changes according to the plan
- Follow all project conventions from CLAUDE.md
- After implementation, run relevant build/test commands to verify:
  - Detect the project's build system from its configuration files (e.g. Makefile, package.json, Cargo.toml, go.mod, pom.xml, build.gradle, mise.toml, .mise.toml, etc.) and CLAUDE.md
  - Use the project's own build and test commands — do not assume a specific package manager or toolchain
- Fix any issues that arise

### Phase 3: Verify
- Walk through each acceptance criterion against the implementation
- For each criterion, confirm it is met or fix what's missing
- Iterate until all criteria are satisfied

### Phase 4: Commit and Update Status
- Update `milestones.md` to mark the milestone as done:
  - Set **Status:** to `done`
  - Check off completed acceptance criteria
- Commit the milestone's changes with a descriptive message: `ship(<spec-name>): <milestone name>`
- Proceed immediately to the next milestone

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
- Commit after each milestone is verified — do not batch commits across milestones
- Only stop to ask the user if you are genuinely blocked (e.g. ambiguous requirement, missing dependency, failing build you cannot fix)
- If a milestone's scope needs to change, stop and suggest updating the spec files first via `/discuss`
- When updating milestones.md, preserve the existing format and only change status and checkboxes
