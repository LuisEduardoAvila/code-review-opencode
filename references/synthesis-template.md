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

## PR Review Template

For GitHub PR reviews, use this template to format the review body:

```markdown
## Code Review Summary

**Decision:** {APPROVE/REQUEST_CHANGES/COMMENT}

### Overview
{1-2 sentence overall assessment of the PR}

### Critical Issues (P0) — Must Fix Before Merge
{For each P0 issue:}
- **{id}**: {description}
  - Location: `path:line`
  - Action: {how to fix}

{If no P0 issues:}
_None_

### Important Issues (P1) — Should Fix
{For each P1 issue:}
- **{id}**: {description}
  - Location: `path:line`
  - Action: {how to fix}

{If no P1 issues:}
_None_

### Minor Issues (P2) — Backlog
{For each P2 issue:}
- **{id}**: {description}

{If no P2 issues:}
_None_

### Verified Good
{For each verified item:}
- ✅ {item}

### Recommendation
{Clear action for the developer}
```

### PR Review Example

```markdown
## Code Review Summary

**Decision:** REQUEST_CHANGES

### Overview
One critical security vulnerability (SQL injection) must be fixed before merge. Two important issues should be addressed. One minor optimization suggestion.

### Critical Issues (P0) — Must Fix Before Merge
- **SEC-001**: SQL injection vulnerability in user search
  - Location: `src/db/queries.ts:42`
  - Action: Use parameterized queries instead of string concatenation

### Important Issues (P1) — Should Fix
- **REF-001**: Missing import for `validateToken`
  - Location: `src/auth/middleware.ts:15`
  - Action: Add `import { validateToken } from './utils'`

- **EDGE-001**: Empty array causes TypeError in reduce
  - Location: `src/utils/aggregator.ts:78`
  - Action: Add initial value to reduce() or check array length

### Minor Issues (P2) — Backlog
- **SIMPL-001**: Complex conditional in processUser
  - Location: `src/services/user.ts:120`
  - Action: Consider using guard clauses for readability

### Verified Good
- ✅ All imports resolve correctly
- ✅ No hardcoded secrets found
- ✅ Error handling covers main paths
- ✅ Test coverage for auth flow

### Recommendation
Fix SQL injection (SEC-001) before merge. Address REF-001 and EDGE-001 in the same PR. SIMPL-001 can be deferred to backlog.
```

### Inline Comment Format

For inline comments, use this format:

```markdown
{SEVERITY_ICON} **{Severity}**: {Brief issue description}

{Detailed explanation}

**Suggestion:**
```{language}
{code suggestion}
```
```

**Example P0:**
```markdown
🚨 **Critical**: SQL injection vulnerability

The query string concatenates user input directly, allowing arbitrary SQL execution:
```ts
const query = `SELECT * FROM users WHERE name = '${name}'`;
```

**Suggestion:**
```ts
const query = 'SELECT * FROM users WHERE name = ?';
db.query(query, [name]);
```
```

**Example P1:**
```markdown
⚠️ **Important**: Missing null check

`user.name` may fail if user is null. This path is reachable when session expires.

**Suggestion:**
```ts
if (!user) {
  throw new AuthenticationError('Session expired');
}
return user.name;
```
```

**Example P2:**
```markdown
💡 **Suggestion**: Consider early return

This nested conditional could be simplified with guard clauses for better readability.

**Suggestion:**
```ts
if (!user) return null;
if (!token) throw new Error('No token');
// rest of function
```
```
