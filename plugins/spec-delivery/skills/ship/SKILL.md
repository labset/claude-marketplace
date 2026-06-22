---
description: Autonomously deliver a spec by orchestrating parallel coding and review subagents per the plan in plan.yaml. Each parallel agent runs in its own worktree with sliced context. Refuses to run without a fresh plan; the only user confirmation gate is in /orchestrate, not here.
argument-hint: <spec path>
disable-model-invocation: true
---

# /ship - Autonomous orchestrator

You execute a spec milestone by milestone according to its `plan.yaml`. You do not plan — `/orchestrate` already did. Your job is to spawn coding and review subagents, enforce the acceptance-criteria gate, merge cleanly, and update state. Work autonomously through the plan; stop only if genuinely blocked.

## Setup

1. Determine the spec:
   - If the user supplied a path (e.g. `/ship .agent/specs/2026-06-22-01-foo`), use it
   - Otherwise list `ls -d .agent/specs/*/` and ask which spec to ship
2. Read `<spec>/milestones.yaml`, `<spec>/requirements.md`, and `<spec>/plan.yaml`. If `plan.yaml` is missing, refuse and tell the user to run `/orchestrate`.
3. **Freshness check**: compute the **structural hash** of `milestones.yaml` (see "Structural hash" at the end of this file) and compare it to `source_hash` in `plan.yaml`. If they differ, refuse and tell the user to re-run `/orchestrate`. Mutable fields (`status`, `acceptance[].verified`) are excluded from the hash so partial runs do not invalidate the plan.
4. Ensure `.agent/worktrees/` is gitignored at the project root. If `.gitignore` exists and lacks the entry, append it. If `.gitignore` does not exist, create it with that single line.
5. Note the parent branch (the user's current branch). All milestone commits land here.

## Execution

Before dispatching anything for a milestone, check its current `status` in `milestones.yaml`. Skip milestones with `status: done` (already shipped, e.g. resuming after an interrupted run) and `status: blocked` (previously failed; the user must intervene before they can ship).

Update `milestones.yaml` immediately after each step that changes state — never batch.

### Step 1: Contracts (sequential)
For each entry in `plan.yaml > contracts`, in declared order:
1. Create the worktree: `git worktree add .agent/worktrees/<spec>/<milestone-id> -b ship/<spec>/<milestone-id>`
2. Dispatch a coding subagent (see "Subagent dispatch") using the milestone's `model`
3. Apply the **AC gate**
4. Dispatch a per-milestone review subagent (contracts default to `review: strict`) using `sonnet`; apply the **review gate** — fixes happen in the existing worktree
5. From the parent branch: `git merge --squash ship/<spec>/<milestone-id>`
6. Update `milestones.yaml`: this milestone's `status: done` and every AC's `verified: true`
7. Stage everything including `milestones.yaml`, then `git commit -m "ship(<spec>): <milestone-name>"`
8. `git worktree remove .agent/worktrees/<spec>/<milestone-id>` and `git branch -D ship/<spec>/<milestone-id>`

If any contracts milestone blocks, stop the entire ship — every downstream wave depends on contracts. Leave the failed worktree in place for user inspection.

### Step 2: Waves (parallel coding → AC gate → wave review → sequential merge → cleanup)
For each wave in `plan.yaml > waves`, in order:

**(a) Parallel coding.** For every milestone in the wave, first create the worktree:

- `git worktree add .agent/worktrees/<spec>/<milestone-id> -b ship/<spec>/<milestone-id>`

Then dispatch all milestones' coding subagents **in a single message with multiple Agent tool uses** so they run concurrently. Each Agent call uses the milestone's `model:` from `plan.yaml`.

**(b) AC gate per milestone.** After all parallel agents return, apply the AC gate to each (retries are serial — see "AC gate"). Milestones that pass continue to review; milestones that block are marked `status: blocked` along with their transitive dependents, and their worktrees are left in place for user inspection.

**(c) Wave review (pre-merge).** If `plan.yaml` declares `review_level` / `review_model` for the wave, dispatch one review subagent over the passing milestones' unmerged branches (`ship/<spec>/<id>`) using `review_model`. Exclude any milestone whose `review:` is `skip` in `milestones.yaml`. Apply the **review gate** — because nothing is merged yet, any retries operate in the existing worktrees and re-running the review is cheap.

**(d) Sequential merge.** For each milestone that passed both AC and review, in any topological order within the wave:
1. From the parent branch: `git merge --squash ship/<spec>/<milestone-id>`
2. Update `milestones.yaml`: set this milestone's `status: done` and every AC's `verified: true`
3. Stage all changes including `milestones.yaml`, then `git commit -m "ship(<spec>): <milestone-name>"`

If a merge collision occurs (orchestrate should have prevented this), stop and surface — do not auto-resolve.

**(e) Cleanup.** For every successfully merged milestone: `git worktree remove .agent/worktrees/<spec>/<milestone-id>` and `git branch -D ship/<spec>/<milestone-id>`. Worktrees of blocked milestones stay in place.

### Step 3: Report
When the plan is fully executed (or every remaining milestone is blocked), summarise:
- Milestones shipped with commit hashes
- Milestones blocked with the failing AC id or review finding
- Suggest `/audit <spec>` if anything blocked

## Subagent dispatch

Use the `Agent` tool with `subagent_type: "claude"`. Do NOT pass `isolation` — we manage worktrees manually. Pass the milestone's `model` (or the wave's `review_model`) as the `model` parameter.

### Coding subagent prompt template

```
You are implementing milestone <id> for spec <spec-name>.

Working directory: <absolute-path-to-worktree>
For all file operations, use absolute paths under this directory. Prefix shell commands with `cd <worktree> &&`.

Requirements (for context):
<paste full requirements.md verbatim>

Milestone:
  id: <id>
  name: <name>
  description: <description>
  type: <feature|contracts>
  touches: <paths from yaml>
  acceptance:
    - id: AC1
      criterion: <criterion>
    - id: AC2
      criterion: <criterion>

Read `CLAUDE.md` in your working directory before starting. Follow all project conventions.

Detect the project's build system from its config files (Makefile, package.json, Cargo.toml, go.mod, pom.xml, build.gradle, mise.toml, .mise.toml) and run the project's own build and test commands to verify your work.

Output budget: ≤150 words plus the structured block below.

End your reply with this exact YAML block (no prose after it):

```yaml
status: done | blocked
summary: <one short paragraph>
acceptance:
  - id: AC1
    verified: true | false
    evidence: <file:line | test name | build-output snippet | reason-if-false>
  - id: AC2
    verified: true | false
    evidence: <as above>
```

If blocked, set `status: blocked`, leave the relevant AC as `verified: false`, and explain in the summary.
```

For a retry, append:
```
RETRY: The previous attempt left these acceptance criteria unverified: <ids>.
Reason given: <prior reply's summary>.
Focus on satisfying these specifically. Same output format.
```

### Review subagent prompt template

```
You are reviewing changes for correctness bugs in spec <spec-name>.

Working directory: <absolute-path-to-parent-branch-checkout>
Parent branch: <parent-branch-name>

Scope: the following unmerged branches against the parent. For each, inspect the diff via `git diff <parent-branch>...<branch>` and read the changed files in the corresponding worktree.

Milestones in scope:
- <id>: <name>   branch: ship/<spec>/<id>   worktree: <absolute-path>
- <id>: <name>   branch: ship/<spec>/<id>   worktree: <absolute-path>

Effort: <low | medium | high>  # low for review: minimal/standard, high for strict

Find correctness bugs, broken invariants, missing error handling at system boundaries (user input, external APIs), and acceptance-criteria violations against the milestone descriptions above. Do NOT flag style, naming, or aesthetic preferences.

Output budget: ≤200 words plus the structured block below.

End your reply with this exact YAML block:

```yaml
verdict: pass | block
findings:
  - milestone: <id-or-wave>
    severity: low | medium | high
    description: <one sentence>
    location: <file:line>
```

Only `severity: high` findings block. Return `verdict: pass` with empty `findings: []` if clean.
```

## AC gate

After a coding subagent returns:
1. Parse the YAML block at the end of its reply
2. If `status: done` and every AC `verified: true` → pass to next step
3. Otherwise:
   - **First attempt**: re-dispatch the coding subagent with the retry suffix, listing unverified AC ids. Same worktree, same model.
   - **Second attempt**: mark the milestone `status: blocked` in `milestones.yaml`, mark transitive dependents `blocked`, leave the worktree intact for user inspection, and continue with the rest of the wave / plan.

## Review gate

After a review subagent returns:
1. Parse the YAML block at the end of its reply
2. `verdict: pass` (or only `low`/`medium` findings) → done
3. `verdict: block` with any `severity: high` finding:
   - **First attempt**: for each implicated milestone, re-dispatch its coding subagent with a retry note quoting the high-severity findings. After fixes, re-merge and re-run the review (one round only).
   - **Second attempt**: mark the implicated milestones `status: blocked` in `milestones.yaml`. The squashed commits stay on the branch — flag in the final report for user inspection.

## Rules

- Refuse to run if `source_hash` mismatches; do not proceed without a fresh plan
- All parallel agents in a wave dispatch in a single message with multiple Agent tool uses
- One Agent call per coding agent; one per review. Never combine.
- Worktrees live at `.agent/worktrees/<spec>/<milestone-id>/`; orchestrator creates and destroys them
- One squashed commit per milestone on the parent branch: `ship(<spec>): <milestone-name>`
- Update `milestones.yaml` immediately after each status-changing step
- Retry budget: 1 per coding agent, 1 per review round-trip. Beyond that, block.
- Never auto-resolve a merge conflict. Surface and stop.
- Do not modify `requirements.md` or `plan.yaml` during ship
- If you genuinely cannot proceed (e.g. plan references a milestone missing from `milestones.yaml`), stop and tell the user

## Structural hash

The `source_hash` in `plan.yaml` excludes mutable fields that `/ship` updates during execution (`milestones[].status` and `milestones[].acceptance[].verified`), so the hash stays stable across partial runs and only invalidates when the user changes the spec shape.

Both `/orchestrate` and `/ship` MUST use this exact algorithm:

```bash
python3 - "<spec>/milestones.yaml" <<'PY'
import sys, yaml, hashlib
d = yaml.safe_load(open(sys.argv[1]))
for m in d.get("milestones", []):
    m.pop("status", None)
    for a in m.get("acceptance", []):
        a.pop("verified", None)
sys.stdout.write(hashlib.sha256(yaml.safe_dump(d, sort_keys=True).encode()).hexdigest())
PY
```

Requires `PyYAML`; if missing, install via `pip install pyyaml` or fall back to an equivalent script in another language using the same field-stripping + sorted-keys YAML serialisation.
