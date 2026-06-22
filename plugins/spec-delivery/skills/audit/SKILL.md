---
description: Audit a spec and its implementation, identifying spec gaps vs implementation gaps with action-tagged findings. Use when the user wants to verify what was built matches what was specified.
argument-hint: <spec path> [--milestone <id>]
disable-model-invocation: true
---

# /audit - Spec and Implementation Audit

You audit a spec against its implementation. Identify spec authoring problems and delivery problems separately, and tag every finding with the action that resolves it.

## Setup

1. Determine the spec:
   - If the user supplied a path (e.g. `/audit .agent/specs/2026-06-22-01-foo`), use it
   - Otherwise run `ls -d .agent/specs/*/` and ask which spec
2. Parse optional `--milestone <id>` flag. When set, scope every phase below to that one milestone — its acceptance criteria, its `touches:` paths, and the requirements it implements.
3. Read `<spec>/requirements.md` and `<spec>/milestones.yaml`. (If only a legacy `milestones.md` exists, tell the user — auditing legacy markdown specs is unsupported.)

## Audit Process

### Phase 1: Spec Consistency (spec-authoring problems)
Check the spec files for internal consistency:
- Every milestone traces back to at least one functional requirement
- Every functional requirement is covered by at least one milestone's acceptance criteria
- No contradictions between requirements and milestone descriptions
- Milestone `status` is consistent with its acceptance `verified` flags (a `done` milestone must have every AC `verified: true`; a `pending` one should have all `verified: false`)
- `depends_on` entries reference existing milestone ids; no cycles
- `touches:` declarations are non-empty and plausible for the described scope

Findings here are spec problems — they go back to `/discuss`.

### Phase 2: Implementation Review (delivery problems)
For each milestone with `status: done` (or only the `--milestone` target):
- Read the actual source files referenced by `touches:` and produced/modified per the milestone description
- For every acceptance criterion, verify it is genuinely satisfied in code — not just claimed via the `verified: true` flag
- Look for partial implementations, TODO comments, placeholder logic, or skipped tests
- Confirm generated code is current with its schema sources

Findings here are delivery problems — they go back to `/ship`.

### Phase 3: Divergence and Quality (mixed)
- **Spec gaps**: requirements present, no milestone implements them
- **Implementation gaps**: milestone marked done, criterion not actually met
- **Scope creep**: code present, no requirement or milestone covers it
- **Contradictions**: behaviour conflicts with the spec
- **Convention violations**: CLAUDE.md or project conventions not followed
- **Quality issues**: inconsistent error handling, missing edge cases, copy-paste duplication

## Action tags

Every finding carries one of:

- **`[FIX-VIA-DISCUSS]`** — the spec itself needs to change (missing requirement, contradiction, bad scope, missing milestone, wrong `touches:`). Resolution: re-run `/discuss <spec-path>`.
- **`[FIX-VIA-SHIP <milestone-id>]`** — the spec is correct but the named milestone was not delivered or was delivered incompletely. Resolution: revert/re-run that milestone via `/ship`.
- **`[FIX-VIA-EDIT <path>]`** — a small, localised correctness issue (typo, off-by-one, missing nil check) that can be fixed directly without re-running a milestone. Resolution: edit the file.

If a finding could plausibly take multiple actions, prefer the smallest one that fully resolves it (`EDIT` < `SHIP` < `DISCUSS`).

## Output Format

Present findings directly to the user. Do not write files.

```
## Audit Report: <spec title> [milestone: <id>]

### Summary
<one-paragraph overall assessment>

### Spec Authoring Issues
| # | Tag | Finding | Location |
|---|-----|---------|----------|
| 1 | [FIX-VIA-DISCUSS] | <finding> | requirements.md / milestones.yaml |

### Delivery Issues
| # | Tag | Finding | Location |
|---|-----|---------|----------|
| 2 | [FIX-VIA-SHIP M2] | <finding> | <file:line> |
| 3 | [FIX-VIA-EDIT internal/api/handlers.go:42] | <finding> | <file:line> |

### Requirement Coverage
| Requirement | Status | Notes |
|-------------|--------|-------|
| FR-1        | Covered / Gap / Partial | <details> |

### Acceptance Criteria Verification
| Milestone | AC | Claimed | Actual | Notes |
|-----------|----|---------|--------|-------|
| M1        | AC1| verified: true | confirmed / unverified | <details> |

### Recommendations
1. <ordered, action-tagged next steps>
```

## Rules

- Be thorough — read every relevant source file referenced by `touches:`, not just the spec
- Be specific — cite file paths and line numbers for every implementation finding
- Be objective — separate observed facts from quality opinions; mark opinions explicitly
- Prioritise by severity within each section (delivery issues that break functionality > convention violations)
- Tag every finding — an untagged finding is incomplete
- If the spec is consistent and the implementation matches, say so plainly — do not manufacture issues
- When scoped via `--milestone`, ignore unrelated milestones entirely; do not comment on them
