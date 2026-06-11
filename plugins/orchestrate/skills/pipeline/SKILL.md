---
description: Run the full spec-delivery pipeline autonomously (ingest, ship, audit) with structured status updates. Use when an orchestrator wants end-to-end delivery.
argument-hint: <name> --from <path>
disable-model-invocation: true
---

# /pipeline - End-to-End Autonomous Delivery Pipeline

You are an autonomous pipeline agent. Your goal is to run the full spec-delivery lifecycle — ingest, ship, audit — in a single session, producing structured status updates at each stage so an orchestrator can track progress.

## Pipeline Stages

### Stage 1: Ingest

1. Follow the `/ingest` skill to create the spec from the provided source material
2. After writing the spec files, update `status.json` with state `"ready"`
3. Output a stage completion signal:
   ```json
   {"stage": "ingest", "result": "success", "spec_path": "<path>"}
   ```

### Stage 2: Ship

1. Follow the `/ship` skill to deliver all milestones
2. After each milestone is committed, update `status.json` with current progress
3. Output a milestone completion signal after each:
   ```json
   {"stage": "ship", "milestone": "M1", "result": "success"}
   ```
4. If you hit a blocker you cannot resolve, write it to `status.json` and output:
   ```json
   {"stage": "ship", "milestone": "M2", "result": "blocked", "reason": "<description>"}
   ```
   Then stop the pipeline — do not proceed to audit.

### Stage 3: Audit

1. Follow the `/audit` skill to verify the implementation against the spec
2. Write the audit findings to `.agent/specs/yyyy-mm-dd-xx-<name>/audit.md`
3. Update `status.json`:
   - If audit passes with no high-severity issues: state `"done"`
   - If audit finds high-severity issues: state `"audit_failed"`
4. Output the final signal:
   ```json
   {"stage": "audit", "result": "pass", "issues": 0}
   ```
   or:
   ```json
   {"stage": "audit", "result": "fail", "issues": 3, "high_severity": 1}
   ```

## Status File Updates

Update `.agent/specs/yyyy-mm-dd-xx-<name>/status.json` at each transition:

```json
{
  "spec": "<name>",
  "path": ".agent/specs/yyyy-mm-dd-xx-<name>",
  "state": "shipping",
  "stage": "ship",
  "milestones": {
    "total": 3,
    "done": 1,
    "in_progress": 1,
    "pending": 1
  },
  "blocked": null,
  "updated_at": "yyyy-mm-ddTHH:MM:SSZ"
}
```

Valid states through the pipeline: `ready` -> `shipping` -> `auditing` -> `done` | `audit_failed` | `blocked`

## Rules

- Run all stages sequentially and autonomously — do not wait for user input between stages
- Output the JSON signal lines as the first thing after completing each stage, before any other commentary
- If the ingest stage fails (e.g. source file not found), stop immediately and output:
  ```json
  {"stage": "ingest", "result": "error", "reason": "<description>"}
  ```
- Always update status.json before outputting the stage signal — the file is the durable record, the signal is ephemeral
- Commit the audit report as a separate commit: `audit(<spec-name>): audit report`
- Follow all rules from the underlying `/ingest`, `/ship`, and `/audit` skills
