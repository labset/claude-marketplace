---
description: Write a structured signal (blocked, unblocked, error) to a spec's status.json. Use when an agent or human needs to flag or resolve a blocker.
argument-hint: <spec path> --blocked <reason> | --unblocked | --error <reason>
disable-model-invocation: true
---

# /signal - Write Structured State Signals

You are a signaling agent. Your goal is to write structured state transitions to a spec's `status.json` so that orchestrators can react to blockers, errors, and resolutions.

## Setup

1. Parse the arguments to determine:
   - The spec path (e.g. `.agent/specs/2026-04-26-01-auth-api`)
   - The signal type: `--blocked`, `--unblocked`, or `--error`
   - The reason (required for `--blocked` and `--error`)
2. Read the current `status.json` from the spec directory

## Signal Types

### --blocked

The agent has hit a blocker it cannot resolve autonomously.

Update `status.json`:
```json
{
  "state": "blocked",
  "blocked": {
    "reason": "<description of what is blocking progress>",
    "milestone": "<which milestone is blocked, e.g. M2>",
    "signaled_at": "yyyy-mm-ddTHH:MM:SSZ"
  },
  "updated_at": "yyyy-mm-ddTHH:MM:SSZ"
}
```

### --unblocked

A blocker has been resolved and work can resume.

Update `status.json`:
```json
{
  "state": "ready",
  "blocked": null,
  "updated_at": "yyyy-mm-ddTHH:MM:SSZ"
}
```

Set state to `ready` if no milestones are in-progress, or `shipping` if the pipeline was mid-flight.

### --error

A fatal error occurred that requires human intervention.

Update `status.json`:
```json
{
  "state": "error",
  "error": {
    "reason": "<description of the error>",
    "stage": "<which stage failed: ingest, ship, audit>",
    "signaled_at": "yyyy-mm-ddTHH:MM:SSZ"
  },
  "updated_at": "yyyy-mm-ddTHH:MM:SSZ"
}
```

## Output

After updating `status.json`, output the signal as JSON:

```json
{"signal": "blocked", "spec": "<name>", "reason": "<reason>"}
```

## Rules

- Always preserve existing fields in status.json that are not being updated (milestones, spec, path)
- Validate that the spec directory and status.json exist before writing — if they don't, output an error signal
- Use ISO 8601 timestamps in UTC
- Output valid JSON only — no markdown wrapping
