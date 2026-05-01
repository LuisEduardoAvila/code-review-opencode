# Synthesis Template

Use this template to synthesize code review outputs into a human-readable summary.

## Template

```markdown
## Code Review Summary

**Decision:** {APPROVE/REQUEST_CHANGES/BLOCK}

### Overview
{overall_assessment from Judge}

### Critical Issues (P0) — Must Fix Before Merge
{For each P0 issue:}
- **{id}**: {description}
  - Action: {action}
  - Source: {source_roles}

### Important Issues (P1) — Should Fix
{For each P1 issue:}
- **{id}**: {description}
  - Action: {action}
  - Source: {source_roles}

### Minor Issues (P2) — Backlog
{For each P2 issue:}
- **{id}**: {description}
  - Action: {action}

### Verified Good
{For each verified item:}
- ✅ {item}

### Recommendation
{recommendation from Judge}
```

## Example Output

```markdown
## Code Review Summary

**Decision:** REQUEST_CHANGES

### Overview
One critical security vulnerability (SQL injection) must be fixed before merge. Two important issues (missing method, empty array edge case) should be addressed. One minor optimization suggestion for large inputs.

### Critical Issues (P0) — Must Fix Before Merge
- **SEC-001**: SQL injection vulnerability in user search
  - Action: Use parameterized queries. CWE-89.
  - Source: Security Probe (S1)

### Important Issues (P1) — Should Fix
- **REF-001**: Method 'getName' does not exist on User class
  - Action: Use 'getFullName' or add 'getName' method
  - Source: Reference Checker (R1)

- **EDGE-001**: Empty array causes TypeError in reduce operation
  - Action: Add initial value to reduce() or check array length
  - Source: Edge Hunter (E1)

### Minor Issues (P2) — Backlog
- **EDGE-002**: Large input batches may cause memory issues
  - Action: Consider implementing batch processing for inputs >10k items

### Verified Good
- ✅ All imports resolve correctly
- ✅ No hardcoded secrets found
- ✅ Error handling covers main paths

### Recommendation
Fix SQL injection (P0) and method reference (P1) before merge. Empty array fix can be in same PR. Large input optimization can be deferred to backlog.
```

## Markdown Formatting

Use markdown formatting for readability:

```markdown
**Code Review Summary**

**Decision:** REQUEST_CHANGES

**Critical Issues (P0)**
• **SEC-001**: SQL injection vulnerability
  Action: Use parameterized queries

**Important Issues (P1)**
• **REF-001**: Method 'getName' does not exist
  Action: Use 'getFullName'

**Verified Good**
✅ All imports resolve
✅ No hardcoded secrets
```

## When to Save Full JSON

Save to `notes/code-review-TIMESTAMP.json` when:
- User requests full details
- More than 10 findings
- Complex issues need context
- User asks to review later

## JSON Structure

When saving JSON, use this structure:

```json
{
  "timestamp": "2026-05-01T12:00:00Z",
  "decision": "REQUEST_CHANGES",
  "roles": {
    "correctness_reviewer": { ... },
    "challenger": { ... },
    "reference_checker": { ... },
    "security_probe": { ... },
    "edge_hunter": { ... },
    "simplifier": { ... }
  },
  "synthesis": {
    "critical_issues": [ ... ],
    "important_issues": [ ... ],
    "minor_issues": [ ... ],
    "verified_good": [ ... ],
    "recommendation": "..."
  }
}
```