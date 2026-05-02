# PR Review Workflow

Step-by-step instructions for performing a GitHub PR review.

## Overview

```
Parse PR → Gather Context → Simulate Merge → Run Roles → Synthesize → Submit Review
```

## Step 1: Parse PR Identifier

From user input, extract:
- Owner (repository owner)
- Repo (repository name)
- PR number
- Optional: base branch override

**Input patterns:**

| Pattern | Example | Extraction |
|---------|---------|------------|
| Number only | `123` | Use current repo, PR #123 |
| Owner/repo#num | `owner/repo#123` | Parse owner, repo, number |
| Full URL | `https://github.com/owner/repo/pull/123` | Parse from URL path |

**Command:**
```bash
# Extract from remote if not specified
git remote get-url origin
# Parse: github.com/owner/repo → owner, repo
```

## Step 2: Gather PR Context

Run these in parallel:

```bash
# PR metadata
gh pr view <number> --json title,body,author,baseRefName,headRefName,files,additions,deletions,commits

# Diff content
gh pr diff <number>

# Commit list
gh api repos/:owner/:repo/pulls/:number/commits
```

**Store:**
- `pr_metadata` — Title, body, author, base/head refs
- `pr_diff` — Raw diff content
- `pr_commits` — Commit history
- `pr_files` — List of changed files

## Step 3: Fetch Base Branch Files

For each changed file in the PR:

1. Get file path from diff
2. Fetch base branch version:
```bash
gh api "repos/:owner/:repo/contents/:path?ref=:baseRef" \
  --jq '.content' | base64 -d > /tmp/base/:path
```

3. Handle special cases:
   - **File created (no base)** → Empty base content
   - **File deleted** → Skip (review deletion rationale only)
   - **Binary file** → Skip (cannot review diff)
   - **Renamed file** → Fetch from old path

**Optimization:** Fetch base files in parallel using multiple bash calls.

## Step 4: Simulate Merged State

For each changed file:

1. Parse diff hunks (lines starting with `@@`)
2. Extract hunk ranges: `@@ -a,b +c,d @@`
3. Apply changes to base content:
   - Remove lines starting with `-`
   - Add lines starting with `+`
   - Keep lines starting with ` ` (context)

4. Track line mapping:
   - **Base line → Merged line** (for understanding original positions)
   - **Merged line → Diff position** (for GitHub comments)

5. Store merged content for each file

**Line mapping state:**
```
{
  "path": "src/utils.ts",
  "base_line_ranges": [[10, 15], [20, 30]],
  "merged_line_ranges": [[10, 17], [20, 28]],
  "diff_positions": {
    "42": 5,   // merged line 42 → diff position 5
    "43": 6,
    "44": 7
  }
}
```

## Step 5: Run Adversarial Roles

Spawn all roles in parallel with merged file content:

```
task(subagent_type="general", description="Correctness Reviewer", prompt="...")
task(subagent_type="general", description="Challenger", prompt="...")
task(subagent_type="general", description="Reference Checker", prompt="...")
task(subagent_type="general", description="Security Probe", prompt="...")
task(subagent_type="general", description="Edge Hunter", prompt="...")
task(subagent_type="general", description="Simplifier", prompt="...")
```

**Key difference from standalone review:**
- Provide **merged file content**, not just the diff
- Include **PR metadata** in prompts (title, body, author intent)
- Note that code is being **added to existing codebase**

**Prompt template for each role:**
```
You are the [ROLE] in a PR code review.

PR Context:
- Title: {pr_title}
- Description: {pr_body}
- Author: {pr_author}
- Base branch: {base_ref}

Your task: Review the MERGED state of these files (what the code will look like after this PR is merged).

[Merged file content]

Reference: references/[role]-prompt.md
```

## Step 6: Synthesis

Run Judge role to:
1. Parse JSON from all role outputs
2. Deduplicate findings (same issue from multiple roles)
3. Prioritize by severity (P0 > P1 > P2)
4. Determine GitHub event (APPROVE/REQUEST_CHANGES/COMMENT)

**Input:**
- All role JSON outputs
- PR metadata
- Line mappings

**Output:**
```json
{
  "event": "APPROVE|REQUEST_CHANGES|COMMENT",
  "body": "Markdown summary",
  "comments": [
    {"path": "src/utils.ts", "position": 5, "body": "Issue description"}
  ]
}
```

## Step 7: Map Findings to Diff Positions

For each finding with a file location:

1. Get merged file line number from finding
2. Look up diff position from line mapping table
3. If position exists, add to `comments[]` array
4. If position is None (unchanged context), add to summary body only

**Filtering rules:**
- P0/P1 → Inline comment if in changed region
- P2 → Inline comment if practical, otherwise summary
- Outside diff → Summary body only

## Step 8: Submit GitHub Review

Build and submit review payload:

```bash
gh api repos/:owner/:repo/pulls/:number/reviews \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -f event="$EVENT" \
  -f body="$BODY" \
  -F comments="$COMMENTS_JSON"
```

**Payload fields:**
- `event` — APPROVE, REQUEST_CHANGES, or COMMENT
- `body` — Markdown summary (max ~64KB)
- `comments[]` — Array of inline comments
  - `path` — File path (relative to repo root)
  - `position` — Diff position (1-indexed from @@ header)
  - `body` — Comment text

**Success:**
```json
{
  "id": 123456,
  "state": "APPROVED",
  "html_url": "https://github.com/owner/repo/pull/123#pullrequestreview-123456"
}
```

## Error Handling

| Error | Action |
|-------|--------|
| `gh` not installed | Error with install instructions |
| Not authenticated | Run `gh auth login` |
| PR not found | Verify PR number and repo |
| Permission denied | Check user has write access |
| API rate limit | Wait and retry |
| Diff too large | Review summary only, no inline comments |
| Base file fetch fails | Continue with partial context |

## Large PR Handling

For PRs with 50+ files:

1. Still review all files (no sampling)
2. Batch role spawns to avoid token limits
3. Summarize findings by category
4. Limit inline comments to high-priority issues (P0/P1 only)
5. Add summary table of files with issues

## Draft PR Handling

Draft PRs receive reviews but:
- Reviews don't block merge (not applicable)
- Note in review body that PR is draft
- Still provide actionable feedback

## Statelessness

Each PR review is independent:
- No storage of previous reviews
- No comparison to prior iterations
- Each run is a fresh analysis

If user wants to re-review after changes:
1. User runs `pr-review` again on the same PR
2. New review submitted as separate review event
3. Previous review remains in PR history