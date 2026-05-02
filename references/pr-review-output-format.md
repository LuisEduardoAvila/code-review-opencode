# PR Review Output Format

GitHub API payload format for submitting PR reviews.

## GitHub PR Review API

**Endpoint:** `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews`

**Documentation:** https://docs.github.com/en/rest/pulls/reviews

## Request Payload

```json
{
  "commit_id": "abc123def456789",
  "event": "APPROVE|REQUEST_CHANGES|COMMENT",
  "body": "Markdown-formatted review summary",
  "comments": [
    {
      "path": "src/utils.ts",
      "position": 5,
      "body": "Inline comment text"
    }
  ]
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `commit_id` | string | No | SHA of commit to review (default: latest) |
| `event` | string | Yes* | Review action: APPROVE, REQUEST_CHANGES, COMMENT |
| `body` | string | Yes** | Markdown summary (*required for REQUEST_CHANGES/COMMENT) |
| `comments` | array | No | Inline comments |

**\* Note:** If `event` is omitted, review is created in PENDING state (not submitted).

### Comment Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | string | Yes | File path relative to repo root |
| `position` | integer | Yes*** | Diff position (1-indexed from @@ header) |
| `body` | string | Yes | Comment text (GitHub Markdown) |
| `line` | integer | No | Line number (alternative to position) |
| `side` | string | No | LEFT or RIGHT (default: RIGHT) |
| `start_line` | integer | No | For multi-line comments |
| `start_side` | string | No | For multi-line comments |

**\*\*\* Important:** Use `position` for reviews, not `line`. `position` is diff-relative.

## gh CLI Syntax

```bash
gh api repos/:owner/:repo/pulls/:number/reviews \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -f event="$EVENT" \
  -f body="$BODY" \
  -F comments="$COMMENTS_JSON"
```

**Note:** Use `-f` for string fields, `-F` for JSON fields (arrays/objects).

## Event Types

| Event | Use Case | Action |
|-------|----------|--------|
| `APPROVE` | No critical issues | Submit approving review |
| `REQUEST_CHANGES` | P0/P1 issues found | Block merge until fixes |
| `COMMENT` | Review feedback, not blocking | Submit feedback without approval |

### Decision Matrix

| P0 Issues | P1 Issues | Event |
|-----------|-----------|-------|
| Yes | Any | `REQUEST_CHANGES` |
| No | 1-2 | `REQUEST_CHANGES` |
| No | 3+ | `REQUEST_CHANGES` |
| No | 0 | `APPROVE` |
| Only P2 | No | `APPROVE` (note P2s in body) |

## Review Body Format

### Summary Template

```markdown
## Code Review Summary

**Decision:** {APPROVE/REQUEST_CHANGES/COMMENT}

### Overview
{1-2 sentence overall assessment}

### Critical Issues (P0) — Must Fix
{List P0 issues or "None"}

### Important Issues (P1) — Should Fix
{List P1 issues or "None"}

### Minor Issues (P2)
{List P2 issues or "None"}

### Verified Good
{What was verified as working}

### Recommendation
{Action for developer}
```

### Example Body

```markdown
## Code Review Summary

**Decision:** REQUEST_CHANGES

### Overview
One critical security vulnerability (SQL injection) must be fixed before merge. Two important issues should be addressed.

### Critical Issues (P0) — Must Fix
- **SEC-001**: SQL injection in user search query

### Important Issues (P1) — Should Fix
- **REF-001**: Missing import for `validateToken`
- **EDGE-001**: Empty array causes crash in `processUsers`

### Minor Issues (P2)
- **SIMPL-001**: Complex conditional could use guard clauses

### Verified Good
✅ Input validation present
✅ Error handling covers main paths
✅ Tests exist for core functionality

### Recommendation
Fix SQL injection (SEC-001) before merge. Address REF-001 and EDGE-001 in the same PR. SIMPL-001 can be deferred.
```

## Comment Format

### Single-Line Comment

```json
{
  "path": "src/auth.ts",
  "position": 12,
  "body": "⚠️ **Important**: Potential null reference\n\n`user.name` may fail if user is null. Consider adding null check:\n```ts\nif (user && user.name) {\n  // ...\n}\n```"
}
```

### Multi-Line Comment

```json
{
  "path": "src/auth.ts",
  "position": 15,
  "start_position": 10,
  "body": "⚠️ **Important**: This block lacks error handling\n\nMultiple API calls without try-catch. Consider wrapping in error boundary."
}
```

### Severity Formatting

| Severity | Prefix | Emoji |
|----------|--------|-------|
| P0 | `🚨 **Critical**` | `:rotating_light:` |
| P1 | `⚠️ **Important**` | `:warning:` |
| P2 | `💡 **Suggestion**` | `:bulb:` |

## Complete Example

### Scenario

PR adds authentication module with:
- 1 SQL injection vulnerability (P0)
- 1 missing import (P1)
- 1 simplification opportunity (P2)

### Payload

```json
{
  "event": "REQUEST_CHANGES",
  "body": "## Code Review Summary\n\n**Decision:** REQUEST_CHANGES\n\n### Overview\nOne critical security vulnerability (SQL injection) must be fixed before merge.\n\n### Critical Issues (P0)\n- **SEC-001**: SQL injection vulnerability in user search\n  \n### Important Issues (P1)\n- **REF-001**: Missing import for `validateToken`\n\n### Minor Issues (P2)\n- **SIMPL-001**: Conditional could use early return\n\n### Recommendation\nFix SQL injection before merge.",
  "comments": [
    {
      "path": "src/auth.ts",
      "position": 42,
      "body": "🚨 **Critical**: SQL injection vulnerability\n\nThe query string concatenates user input directly:\n```ts\nconst query = `SELECT * FROM users WHERE name = '${name}'`;\n```\n\nUse parameterized queries:\n```ts\nconst query = 'SELECT * FROM users WHERE name = ?';\ndb.query(query, [name]);\n```"
    },
    {
      "path": "src/auth.ts",
      "position": 15,
      "body": "⚠️ **Important**: Missing import\n\n`validateToken` is called but not imported. Add:\n```ts\nimport { validateToken } from './utils';\n```"
    },
    {
      "path": "src/auth.ts",
      "position": 78,
      "body": "💡 **Suggestion**: Simplify conditional\n\nConsider using early return pattern:\n```ts\nif (!user) return null;\nif (!token) throw new Error('No token');\n// rest of function\n```"
    }
  ]
}
```

### Submission

```bash
gh api repos/acme/app/pulls/123/reviews \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -f event='REQUEST_CHANGES' \
  -f body='## Code Review Summary...' \
  -F comments='[{"path":"src/auth.ts","position":42,"body":"🚨 **Critical**..."}]'
```

## Response

**Success (200 OK):**

```json
{
  "id": 12345678,
  "node_id": "PRR_...",
  "user": {
    "login": "opencode-bot"
  },
  "body": "## Code Review Summary...",
  "state": "CHANGES_REQUESTED",
  "html_url": "https://github.com/acme/app/pull/123#pullrequestreview-12345678",
  "submitted_at": "2026-05-02T12:00:00Z"
}
```

**Error (403 Forbidden):**

```json
{
  "message": "Resource not accessible by integration",
  "documentation_url": "https://docs.github.com/rest..."
}
```

## Error Handling

| Status Code | Meaning | Resolution |
|-------------|---------|------------|
| 200 | Success | Review submitted |
| 403 | Forbidden | Check write permissions |
| 404 | Not found | Verify PR exists |
| 422 | Validation failed | Check payload format |
| 429 | Rate limited | Wait and retry |

### Rate Limits

GitHub API rate limits:
- 5,000 requests/hour (authenticated)
- Primary rate limit: 1,000 requests/hour per repository

If rate limited, wait for reset (check `X-RateLimit-Reset` header).

## Best Practices

### Comment Limits

GitHub recommends:
- Max 100 comments per review
- Max 10,000 characters per comment
- Max 64KB for total body

**If too many findings:**
1. Prioritize P0/P1 for inline comments
2. Summarize P2 in body
3. Group related findings

### Markdown in Comments

Supported:
- Headers (`##`, `###`)
- Lists (`-`, `1.`)
- Code blocks (```lang```)
- Bold (`**text**`)
- Links (`[text](url)`)

GitHub renders comments as GFM (GitHub Flavored Markdown).

### Draft PRs

For draft PRs:
- Still submit review
- Note "PR is draft" in body
- Use `COMMENT` event (not blocking)

```json
{
  "event": "COMMENT",
  "body": "This PR is a draft. Here's initial feedback for when you're ready.\n\n## Code Review Summary\n..."
}
```