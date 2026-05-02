# PR Context Gathering

Commands and data extraction patterns for GitHub PR reviews.

## Required Tools

- `gh` CLI (GitHub CLI)
- `git` (for remote parsing)
- `base64` (for decoding file content)

## Installation Check

```bash
# Verify gh is installed
which gh && gh --version

# Check authentication
gh auth status
```

If not authenticated:
```bash
gh auth login
```

## PR Identification

### From Current Directory

```bash
# Get owner/repo from git remote
git remote get-url origin
# Outputs: git@github.com:owner/repo.git or https://github.com/owner/repo.git

# Parse owner/repo
git remote get-url origin | sed -E 's#.*github.com[/:]([^/]+)/([^.]+).*#\1/\2#'
```

### From PR Input

| Input Pattern | Parse Method |
|---------------|--------------|
| `123` | Use current repo, extract PR number |
| `owner/repo#123` | Split on `#`, then split owner/repo |
| `https://github.com/owner/repo/pull/123` | Parse URL path |

**Extraction commands:**
```bash
# From number only
PR_NUM=123

# From owner/repo#123
echo "owner/repo#123" | sed 's/#/ /' | read OWNER_REPO PR_NUM
OWNER=$(echo $OWNER_REPO | cut -d'/' -f1)
REPO=$(echo $OWNER_REPO | cut -d'/' -f2)

# From URL
echo "https://github.com/owner/repo/pull/123" | sed -E 's#.*github.com/([^/]+)/([^/]+)/pull/([0-9]+)#\1 \2 \3#'
```

## GitHub API Commands

### PR Metadata

```bash
gh pr view <number> --json title,body,author,baseRefName,headRefName,files,additions,deletions,commits,state,draft
```

**Output fields:**
```json
{
  "title": "Add user authentication",
  "body": "Implements OAuth2 login flow...",
  "author": {"login": "developer"},
  "baseRefName": "main",
  "headRefName": "feature/auth",
  "files": [
    {"path": "src/auth.ts", "additions": 50, "deletions": 5}
  ],
  "additions": 150,
  "deletions": 20,
  "commits": [{"sha": "abc123", "message": "Add OAuth"}],
  "state": "OPEN",
  "draft": false
}
```

### PR Diff

```bash
gh pr diff <number>
```

**Output format:** Unified diff

```
diff --git a/src/auth.ts b/src/auth.ts
index 1234567..abcdefg 100644
--- a/src/auth.ts
+++ b/src/auth.ts
@@ -10,5 +10,15 @@ import { User } from './user';
     const user = await getUser();
+    const token = generateToken(user);
+    if (!token) {
+        throw new Error('Token generation failed');
+    }
     return user;
```

### Commit History

```bash
gh api repos/:owner/:repo/pulls/:number/commits
```

**Output:**
```json
[
  {
    "sha": "abc123def456",
    "commit": {
      "message": "Add OAuth implementation",
      "author": {"name": "Developer", "email": "dev@example.com"}
    }
  }
]
```

### Base Branch Files

For each file in the PR, fetch the base version:

```bash
gh api "repos/:owner/:repo/contents/:path?ref=:baseRef"
```

**Output:**
```json
{
  "name": "auth.ts",
  "path": "src/auth.ts",
  "content": "Ly8gQmFzZSB2ZXJzaW9u...",  // Base64 encoded
  "encoding": "base64"
}
```

Decode:
```bash
gh api "repos/:owner/:repo/contents/:path?ref=:baseRef" --jq '.content' | base64 -d
```

**Error handling:**
```bash
# Check if file exists in base
gh api "repos/:owner/:repo/contents/:path?ref=:baseRef" 2>&1
# 404 = file doesn't exist (new file in PR)
```

### Linked Issues

If PR body references issues (e.g., "Fixes #123"):

```bash
gh api repos/:owner/:repo/issues/123
```

Extract from PR body:
```bash
echo "$PR_BODY" | grep -oE '#[0-9]+' | sed 's/#//'
```

## Diff Structure

### Unified Diff Format

```
diff --git a/path/file.ts b/path/file.ts
index 1234567..abcdefg 100644
--- a/path/file.ts
+++ b/path/file.ts
@@ -10,5 +10,10 @@ context line
 unchanged line
-removed line
+added line
 unchanged line
```

**Key markers:**
- `diff --git` — File header
- `---` / `+++` — Old/new file paths
- `@@ -a,b +c,d @@` — Hunk header (old range, new range)
- `-` (minus) — Removed line
- `+` (plus) — Added line
- ` ` (space) — Unchanged context line

### Hunk Range Parsing

```
@@ -10,5 +10,10 @@
   ^  ^   ^   ^
   |  |   |   +--- New file: 10 lines starting at line 10
   |  |   +------- Old file: 5 lines starting at line 10
   |  +----------- Separator
   +-------------- Old file marker
```

## File Lists

### Changed Files Only

```bash
gh pr view <number> --json files --jq '.files[].path'
```

### Filter by Type

```bash
# Only TypeScript files
gh pr view <number> --json files --jq '.files[].path' | grep '\.ts$'

# Exclude vendor/third-party
gh pr view <number> --json files --jq '.files[].path' | grep -v 'vendor/'
```

### File Statistics

```bash
gh pr view <number> --json files --jq '.files[] | "\(.path): +\(.additions)/-\(.deletions)"'
```

## Performance Considerations

### Parallel Fetching

Fetch base files in parallel:

```bash
# Using GNU parallel or xargs
gh pr view <number> --json files --jq '.files[].path' | \
  xargs -I{} -P8 gh api "repos/:owner/:repo/contents/{}?ref=:baseRef" --jq '.content' | \
  base64 -d
```

### Rate Limits

GitHub API rate limits:
- Authenticated: 5,000 requests/hour
- Unauthenticated: 60 requests/hour (not applicable for this use case)

**Rate limit check:**
```bash
gh api rate_limit
```

### Large Files

Skip files over 1MB:

```bash
gh api "repos/:owner/:repo/contents/:path?ref=:baseRef" --jq '.size'
# If size > 1000000, skip detailed review
```

## Data Structures

### PR Context Object

```json
{
  "pr": {
    "number": 123,
    "title": "Add OAuth authentication",
    "body": "Implements login flow...",
    "author": "developer",
    "base_ref": "main",
    "head_ref": "feature/auth",
    "state": "OPEN",
    "draft": false,
    "additions": 150,
    "deletions": 20,
    "files": ["src/auth.ts", "src/user.ts", "tests/auth.test.ts"]
  },
  "commits": [
    {"sha": "abc123", "message": "Add OAuth", "author": "developer"}
  ],
  "linked_issues": [
    {"number": 45, "title": "Users can't log in"}
  ]
}
```

### File State Object

```json
{
  "path": "src/auth.ts",
  "base_content": "original code...",
  "merged_content": "final code after merge...",
  "diff_hunks": [
    {
      "old_start": 10,
      "old_lines": 5,
      "new_start": 10,
      "new_lines": 10,
      "changes": [...]
    }
  ],
  "line_map": {
    "42": {"diff_position": 5, "type": "added"},
    "43": {"diff_position": 6, "type": "context"}
  }
}
```

## Error Messages

| Error | Cause | Resolution |
|-------|-------|------------|
| `gh: command not found` | CLI not installed | Install `gh` via package manager |
| `not logged in` | No auth token | Run `gh auth login` |
| `404 Not Found` | PR doesn't exist | Verify PR number and repo |
| `403 Forbidden` | No access | Check repository permissions |
| `422 Validation Failed` | Invalid payload | Check API parameters |
| `rate limit exceeded` | Too many requests | Wait and retry |