---
description: Plan a spec's execution before shipping. Reads milestones.yaml, builds a dependency DAG, computes parallel execution waves, picks models, estimates token cost, and writes plan.yaml. Spawns no agents — /ship refuses to run without a fresh plan.
argument-hint: <spec path>
disable-model-invocation: true
---

# /orchestrate - Plan execution waves before shipping

You produce a `plan.yaml` from a spec's `milestones.yaml`. You do NOT spawn coding or review agents — that is `/ship`'s job. Your output is a single artifact the user confirms once, after which `/ship` executes autonomously.

## Setup

1. Determine the spec:
   - If the user supplied a path (e.g. `/orchestrate .agent/specs/2026-06-22-01-foo`), use it
   - Otherwise run `ls -d .agent/specs/*/` and ask which spec to plan
2. Read `<spec>/milestones.yaml`. If missing or empty, stop and tell the user to run `/discuss` first.

## Planning Process

### Phase 1: Validate
Reject the spec with clear errors if any of these fail:
- Every milestone has `id`, `name`, `status`, `depends_on`, `model`, `review`, `acceptance`
- Every `depends_on` entry references an existing milestone id
- No dependency cycles
- `model` is one of `haiku` / `sonnet` / `opus`
- `review` is one of `skip` / `minimal` / `standard` / `strict`

Skip milestones with `status: done` from the plan. Warn about `status: blocked` milestones (they and their transitive dependents are excluded).

### Phase 2: Build the DAG
- **Contracts**: collect all milestones with `type: contracts`. These run sequentially (in dependency order) before any parallel wave begins.
- **Waves**: topologically sort the remaining milestones. Wave N contains every milestone whose `depends_on` entries are all in earlier waves (or in the contracts block).

### Phase 3: Detect Parallel Conflicts
Within each wave, two milestones cannot safely run in parallel if their `touches:` lists overlap (any shared path, or one is a path prefix of the other). A milestone with missing or empty `touches:` is treated as "may conflict with anything" and is isolated.

Auto-resolve by greedily splitting the wave into conflict-free sub-waves while preserving dependency order. Record every split in `conflicts_resolved:` in the plan.

Be conservative — prefer over-sequentialising to risking a merge collision.

### Phase 4: Estimate Cost
Use these order-of-magnitude heuristics per milestone:

| Component | Estimated tokens |
|-----------|------------------|
| Coding agent: `haiku`   | ~30k  |
| Coding agent: `sonnet`  | ~70k  |
| Coding agent: `opus`    | ~150k |
| Wave review (`minimal`/`standard`, runs on Haiku low) | ~30k per wave |
| Wave review (`strict`, runs on Sonnet)                | ~70k per wave |
| Possible retry budget (1 per agent + 1 per review)    | add ~25% headroom |

Sum across contracts + all waves + reviews; round to the nearest 10k. State the heuristic explicitly when presenting the estimate so the user knows it is approximate.

### Phase 5: Present and Confirm
Display the plan as a structured tree:

```
Plan for spec: <name>

CONTRACTS (sequential)
  M0 — <name>                  [model: haiku, review: strict]

WAVE 1 (parallel, N agents)
  ├─ M1 — <name>               [model: sonnet, review: standard]
  └─ M2 — <name>               [model: sonnet, review: standard]

WAVE 2 (parallel, 1 agent)
  └─ M3 — <name>               [model: opus,   review: strict]

CONFLICTS RESOLVED
  • W1 → W1, W2 because M2 and M3 both touch internal/api/

ESTIMATED COST (order of magnitude, +25% retry headroom)
  contracts: ~40k
  wave 1:    ~210k
  wave 2:    ~220k
  total:     ~470k tokens
```

Ask the user to confirm before writing.

### Phase 6: Write plan.yaml
On confirmation:
1. Compute `source_hash`: `shasum -a 256 "<spec>/milestones.yaml" | awk '{print $1}'`
2. Write `<spec>/plan.yaml` per the schema below
3. Tell the user that `/ship <spec>` is now ready to run

Always overwrite any existing `plan.yaml` with the freshly computed one; note in the response if waves or estimates changed from a prior plan.

## plan.yaml schema

```yaml
spec: <spec-name>
source_hash: <sha256-of-milestones.yaml>
generated_at: <iso8601-timestamp>

contracts:
  - id: M0
    model: haiku
    review: strict

waves:
  - id: W1
    milestones:
      - id: M1
        model: sonnet
      - id: M2
        model: sonnet
    review_level: standard
    review_model: haiku

  - id: W2
    milestones:
      - id: M3
        model: opus
    review_level: strict
    review_model: sonnet

conflicts_resolved:
  - reason: M2 and M3 both touch internal/api/
    split: [W1, W2]

estimated_tokens:
  total: 470000
  contracts: 40000
  waves:
    - id: W1
      tokens: 210000
    - id: W2
      tokens: 220000
```

Field notes:
- `review_level` per wave = the highest declared level across the wave's milestones, ignoring any with `review: skip`
- `review_model` per wave = `sonnet` if any milestone in the wave is `review: strict`, otherwise `haiku`
- If every milestone in a wave is `review: skip`, omit `review_level` and `review_model`

## Rules

- You do not spawn coding or review agents — that is exclusively `/ship`'s job
- Always recompute `source_hash` before writing
- Refuse to write the plan if `milestones.yaml` is invalid; surface the validation errors instead
- If the plan is empty (everything is already `done`), tell the user there is nothing to ship
- Be conservative on conflicts — over-sequentialising is cheaper than recovering from a merge collision
- Token estimates are heuristic, not enforced; this skill does not abort or limit execution
