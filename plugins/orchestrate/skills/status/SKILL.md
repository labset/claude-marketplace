---
description: Report machine-readable status of one or all specs. Use when an orchestrator or human needs to check delivery progress.
argument-hint: <spec path or blank for all>
disable-model-invocation: true
---

# /status - Machine-Readable Spec Status

You are a status reporter. Your goal is to read spec files and produce structured, machine-readable status output that orchestrators can parse.

## Setup

1. Determine which specs to report on:
   - If the user provides arguments (e.g. `/status .agent/specs/2026-04-26-01-generic-crud-api`), report on that spec
   - Otherwise, run `ls -d .agent/specs/*/` to list all spec directories and report on all of them
2. For each spec, read `milestones.md` and `status.json` (if it exists)

## Status Extraction

For each spec:

1. Parse `milestones.md` to determine:
   - Total number of milestones
   - Count by status: done, in-progress, pending
   - The name and status of each milestone
   - Whether acceptance criteria checkboxes are checked
2. Read `status.json` if it exists for blocked state and last update time
3. Determine the overall spec state:
   - `ready` — all milestones are pending, no work has started
   - `in_progress` — at least one milestone is in-progress or done, but not all are done
   - `done` — all milestones are done
   - `blocked` — a `blocked` entry exists in status.json

## Output

Output a single JSON document to the user. For a single spec:

```json
{
  "spec": "<name>",
  "path": ".agent/specs/yyyy-mm-dd-xx-<name>",
  "state": "in_progress",
  "milestones": {
    "total": 4,
    "done": 2,
    "in_progress": 1,
    "pending": 1,
    "details": [
      { "id": "M1", "name": "<name>", "status": "done" },
      { "id": "M2", "name": "<name>", "status": "done" },
      { "id": "M3", "name": "<name>", "status": "in-progress" },
      { "id": "M4", "name": "<name>", "status": "pending" }
    ]
  },
  "blocked": null,
  "updated_at": "yyyy-mm-ddTHH:MM:SSZ"
}
```

For multiple specs, output a JSON array.

After outputting the JSON, also update (or create) `status.json` in each spec directory with the current state.

## Rules

- Output valid JSON only — no markdown wrapping, no commentary before or after the JSON
- Derive status from milestones.md as the source of truth, not from a stale status.json
- Always update status.json after reporting so it stays in sync
- If a spec directory has no milestones.md, report its state as `"invalid"` with a note
