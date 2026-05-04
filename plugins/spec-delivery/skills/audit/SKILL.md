---
description: Conduct a thorough audit of a spec and its implementation, identifying discrepancies, gaps, and quality issues. Use when the user wants to verify what was built matches what was specified.
argument-hint: <spec path>
disable-model-invocation: true
---

# /audit - Spec and Implementation Audit

You are conducting a thorough audit of a spec and its implementation. Your goal is to identify discrepancies, divergences, gaps, and issues between what was specified and what was built.

## Setup

1. Determine which spec to audit:
   - If the user provides arguments (e.g. `/audit specs/2026-04-26-01-generic-crud-api`), use the specified directory
   - Otherwise, run `ls -d specs/*/` to list existing spec directories and ask the user which one to audit
2. Read `specs/yyyy-mm-dd-xx-<name>/requirements.md` and `specs/yyyy-mm-dd-xx-<name>/milestones.md`

## Audit Process

### Phase 1: Spec Consistency

Review the spec files for internal consistency:
- Do all milestones trace back to at least one requirement?
- Are all requirements covered by at least one milestone's acceptance criteria?
- Are there any contradictions between requirements and milestone descriptions?
- Are milestone statuses consistent with their acceptance criteria checkboxes?

### Phase 2: Implementation Review

For each milestone marked as done, verify the implementation:
- Read the actual source files that were created or modified
- Check each acceptance criterion against the code — is it genuinely satisfied?
- Look for partial implementations, TODO comments, or placeholder logic
- Verify generated code is up to date with the schema definitions

### Phase 3: Divergence Detection

Identify where the implementation diverges from the spec:
- Features implemented but not specified (scope creep)
- Requirements specified but not implemented (gaps)
- Behaviour that contradicts the spec (bugs or misinterpretations)
- Conventions from CLAUDE.md that were not followed

### Phase 4: Quality Review

Assess the implementation quality:
- Are error cases handled consistently across all handlers?
- Is the code DRY — are there patterns that should be shared but aren't?
- Are there edge cases not covered (empty inputs, boundary conditions)?
- Is the code structured according to project conventions?

## Output Format

Present findings as a structured audit report. Do NOT write files — present the report directly to the user.

```
## Audit Report: <spec title>

### Summary
<one paragraph overall assessment>

### Spec Consistency
- [PASS/ISSUE] <finding>

### Requirement Coverage
| Requirement | Status | Notes |
|-------------|--------|-------|
| FR-1        | Covered / Gap / Partial | <details> |

### Milestone Verification
| Milestone | Criterion | Status | Notes |
|-----------|-----------|--------|-------|
| M1        | <criterion> | Verified / Issue | <details> |

### Divergences
- [SCOPE CREEP] <description>
- [GAP] <description>
- [CONTRADICTION] <description>

### Quality Issues
- [SEVERITY: low/medium/high] <description>

### Recommendations
1. <actionable recommendation>
```

## Rules

- Be thorough — read every relevant source file, do not rely on assumptions
- Be specific — cite file paths and line numbers for every finding
- Be objective — distinguish facts from opinions
- Prioritise findings by severity
- If the spec is fully consistent and the implementation is correct, say so — do not manufacture issues
