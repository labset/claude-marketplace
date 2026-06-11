---
description: Review completed or in-progress specs to identify deferred items, gaps, and opportunities for future work. Use when the user wants to plan what to build next.
argument-hint: <spec path or blank for all>
disable-model-invocation: true
---

# /review - Spec Review and Future Work Recommendations

You are reviewing completed or in-progress specs to identify deferred items, gaps, and opportunities for future work that should be discussed and turned into new specs.

## Setup

1. Determine which specs to review:
   - If the user provides arguments (e.g. `/review .agent/specs/2026-04-29-01-protoc-backend-codegen`), review that spec
   - Otherwise, run `ls -d .agent/specs/*/` to list all spec directories and review all of them
2. For each spec, read `requirements.md` and `milestones.md`

## Review Process

### Phase 1: Gather Deferred Items

For each spec, collect:
- Requirements marked as deferred (DFR-* items)
- Milestone acceptance criteria marked as deferred
- NFRs that are partially implemented with notes
- Out-of-scope items that may now be in scope given what was built

### Phase 2: Identify Implicit Gaps

Look beyond what's explicitly deferred:
- Read the implementation to find TODOs, hardcoded values, or placeholder logic
- Check if conventions in CLAUDE.md are not yet followed by generated or hand-written code
- Look for patterns that work for the sample model but may break for edge cases
- Identify missing test coverage, observability, or operational concerns

### Phase 3: Group into Themes

Organise findings into coherent themes that could each become a new spec:
- Group related deferred items together
- Distinguish quick fixes from substantial new work
- Identify dependencies between themes

### Phase 4: Recommend Future Work

Present findings as a prioritised list of recommended future specs. For each recommendation:
- Give it a short name (suitable for `/discuss <name>`)
- Explain what it would cover
- List which deferred items and gaps it addresses
- Estimate scope: small (1-2 milestones), medium (3-4 milestones), large (5+ milestones)
- Note any dependencies on other recommendations

## Output Format

Present findings directly to the user. Do NOT write files.

```
## Spec Review: <title or "All Specs">

### Deferred Items

| Source Spec | Item | Description |
|-------------|------|-------------|
| <spec name> | DFR-1 | <description> |

### Implicit Gaps

- [<spec name>] <description of gap found in implementation>

### Recommended Future Specs

#### 1. <short-name> (scope: small/medium/large)

> <one-line summary>

**Covers:**
- DFR-1: <from spec X>
- Gap: <implicit gap found>

**Dependencies:** none / <other recommendation>

**Suggested command:** `/discuss <short-name>`

---
```

## Rules

- Be thorough — read implementation files, not just spec files
- Be practical — only recommend work that delivers clear value
- Be specific — cite which deferred items and gaps each recommendation addresses
- Prioritise by impact: correctness issues first, then developer experience, then nice-to-haves
- Do not recommend work that is explicitly out of scope unless circumstances have changed
- Group small related fixes into a single recommendation rather than listing them individually
