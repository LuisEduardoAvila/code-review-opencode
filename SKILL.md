---
name: code-review
description: "Multi-perspective adversarial code review with specialized roles (Challenger, Reference Checker, Security Probe, Edge Hunter, Judge). Use when reviewing code for bugs, security issues, maintainability, or before merging. Triggers on 'code-review', 'review this code', 'adversarial code review', 'security review'. Modes include quick, verify, security, quality, full, and pr-review (experimental)."
metadata:
  pr-review-mode: experimental
---

# Code Review

Multi-perspective adversarial code review with configurable roles.

## Overview

Unlike traditional code review that finds bugs, adversarial code review **actively tries to break the code**. Each role attacks from a different angle — assumptions, references, security, edge cases.

## Context Requirements (CRITICAL)

**Reviews without context produce generic feedback.** Before running any mode, provide:

1. **Requirements/Spec** — What is this code supposed to do?
2. **Coding Conventions** — Style guide, patterns, anti-patterns
3. **Related Files** — Dependencies, callers, tests

**Minimum context by mode:**

| Mode | Minimum Context Required |
|------|--------------------------|
| `quick` | File path or snippet |
| `verify` | Requirements + file path |
| `security` | Architecture overview + file path |
| `quality` | Coding conventions + file path |
| `full` | Requirements + conventions + architecture |

## Risk-Based Mode Selection

Choose mode based on code risk:

| Risk Level | Characteristics | Recommended Mode |
|------------|-----------------|-------------------|
| **Low** | Internal tool, no PII, no external API | `quick` or `quality` |
| **Medium** | Customer-facing, moderate complexity | `verify` (default) |
| **High** | Auth, payment, PII, security-critical | `security` or `full` |
| **Critical** | Core infrastructure, data pipeline | `full` |

**Risk assessment questions:**

1. Does this code handle authentication or authorization?
2. Does it process payment or financial data?
3. Does it store or transmit PII?
4. Is it a core infrastructure component?
5. What's the blast radius if it fails?

**Answer "yes" to any? Use `security` or `full` mode.**

## Pre-Review Checklist

**Before running roles, verify:**

- [ ] Context provided (requirements, conventions)
- [ ] File type detected (backend, frontend, infrastructure)
- [ ] Language/framework identified
- [ ] Test coverage checked (does code have tests?)

**Quick checks that roles skip:**

| Check | Run First |
|-------|-----------|
| Linting errors | `eslint`/`ruff`/`flake8` |
| Format issues | `prettier`/`black`/`gofmt` |
| Type errors | `tsc`/`mypy`/`pyright` |
| Basic tests | Test suite |

**Roles focus on what tools can't check:**
- Intent verification (Correctness)
- Assumption challenges (Challenger)
- Reference existence (Reference Checker)
- Security vulnerabilities (Security Probe)
- Edge cases (Edge Hunter)
- Complexity reduction (Simplifier)

## Review Roles

| Role | Focus | Approach |
|------|-------|----------|
| **Correctness Reviewer** | Intent verification | "Does this code do what it's supposed to do?" |
| **Challenger** | Attack assumptions | "What if this API changes? What if this dependency is removed? What's the blast radius?" |
| **Reference Checker** | Verify existence | "Does `utils.parse()` exist? Is this import path correct? Does this object have that method?" |
| **Security Probe** | Exploit vectors | "Can this input be injected? What's the privilege escalation path? Where does data flow?" |
| **Edge Hunter** | Boundary cases | "Empty arrays? Null objects? Concurrent access? Memory pressure? Offline?" |
| **Simplifier** | Reduce complexity | "Is this abstraction necessary? Could this be simpler? What's the complexity cost?" |
| **Judge** | Synthesis | Weigh severity, prioritize fixes, balance trade-offs |

## Modes

| Mode | Roles | Use Case | Invocation |
|------|-------|----------|------------|
| `quick` | Correctness Reviewer + Challenger | Pre-commit sanity check | `"code-review mode: quick"` |
| `verify` | Correctness Reviewer + Challenger + Reference Checker | CI/CD gate | `"code-review mode: verify"` |
| `security` | Correctness Reviewer + Challenger + Security Probe | Before deploy | `"code-review mode: security"` |
| `quality` | Correctness Reviewer + Simplifier | Code quality review | `"code-review mode: quality"` |
| `full` | All seven roles | PR review | `"code-review mode: full"` |
| `pr-review` | All roles + GitHub integration | GitHub PR review (experimental) | `"pr-review [PR]"` |

**Default mode:** `verify` (Correctness Reviewer + Challenger + Reference Checker) — correctness, adversarial, and existence checks.

**Quality mode:** Use when code works but you want to reduce technical debt and improve maintainability.

**Note:** Correctness Reviewer runs in all modes because **correctness is not optional**.

## Quick Commands

### Quick Mode (Correctness Reviewer + Challenger)

Spawn both in parallel:

```
task(
  subagent_type="general",
  description="Correctness Reviewer: Verify intent",
  prompt="You are the Correctness Reviewer role. Verify that code does what it's supposed to do. Reference: references/correctness-reviewer-prompt.md\n\nCode: [PATH or snippet]"
)

task(
  subagent_type="general",
  description="Challenger: Attack assumptions",
  prompt="You are the Challenger role. Attack this code's assumptions. What if APIs change? Dependencies removed? Edge cases? Find weaknesses. Reference: references/challenger-prompt.md\n\nCode: [PATH or snippet]"
)
```

### Verify Mode (Correctness Reviewer + Challenger + Reference Checker) — DEFAULT

Spawn all three in parallel:

```
task(
  subagent_type="general",
  description="Correctness Reviewer: Verify intent",
  prompt="You are the Correctness Reviewer role. Verify code does what it's supposed to do. Check intent, requirements, business rules. Reference: references/correctness-reviewer-prompt.md\n\nCode: [PATH]"
)

task(
  subagent_type="general",
  description="Challenger: Attack assumptions",
  prompt="You are the Challenger role. Attack assumptions. APIs change, dependencies fail, edge cases. Reference: references/challenger-prompt.md\n\nCode: [PATH]"
)

task(
  subagent_type="general",
  description="Reference Checker: Verify existence",
  prompt="You are the Reference Checker role. Verify all imports, function calls, object methods exist. Check signatures match. Reference: references/reference-checker-prompt.md\n\nCode: [PATH]"
)
```

### Security Mode (Correctness Reviewer + Challenger + Security Probe)

Spawn all three in parallel:

```
task(
  subagent_type="general",
  description="Correctness Reviewer: Verify intent",
  prompt="You are the Correctness Reviewer role. Verify code does what it's supposed to do. Reference: references/correctness-reviewer-prompt.md\n\nCode: [PATH]"
)

task(
  subagent_type="general",
  description="Challenger: Attack assumptions",
  prompt="You are the Challenger role. Attack assumptions. Reference: references/challenger-prompt.md\n\nCode: [PATH]"
)

task(
  subagent_type="general",
  description="Security Probe: Find vulnerabilities",
  prompt="You are the Security Probe role. Find exploit vectors. Injection, auth bypass, data exposure, privilege escalation. Reference: references/security-probe-prompt.md\n\nCode: [PATH]"
)
```

### Quality Mode (Correctness Reviewer + Simplifier)

Spawn both in parallel:

```
task(
  subagent_type="general",
  description="Correctness Reviewer: Verify intent",
  prompt="You are the Correctness Reviewer role. Verify intent and requirements. Does code do what it's supposed to? Reference: references/correctness-reviewer-prompt.md\n\nCode: [PATH]"
)

task(
  subagent_type="general",
  description="Simplifier: Reduce complexity",
  prompt="You are the Simplifier role. Find complexity reduction opportunities. Unnecessary abstractions, over-engineering, duplication. Reference: references/simplifier-prompt.md\n\nCode: [PATH]"
)
```

### Full Mode (All Seven Roles)

Spawn all seven in parallel (Judge runs after synthesis):

```
task(subagent_type="general", description="Correctness Reviewer: Verify intent", prompt="You are the Correctness Reviewer role. Reference: references/correctness-reviewer-prompt.md\n\nCode: [PATH]")

task(subagent_type="general", description="Challenger: Attack assumptions", prompt="You are the Challenger role. Reference: references/challenger-prompt.md\n\nCode: [PATH]")

task(subagent_type="general", description="Reference Checker: Verify existence", prompt="You are the Reference Checker role. Reference: references/reference-checker-prompt.md\n\nCode: [PATH]")

task(subagent_type="general", description="Security Probe: Find vulnerabilities", prompt="You are the Security Probe role. Reference: references/security-probe-prompt.md\n\nCode: [PATH]")

task(subagent_type="general", description="Edge Hunter: Find boundary cases", prompt="You are the Edge Hunter role. Reference: references/edge-hunter-prompt.md\n\nCode: [PATH]")

task(subagent_type="general", description="Simplifier: Reduce complexity", prompt="You are the Simplifier role. Reference: references/simplifier-prompt.md\n\nCode: [PATH]")
```

Note: Judge runs **after** synthesis, not in parallel with other roles.

## PR Review Mode (Experimental)

> ⚠️ **Experimental**: This mode is under active development. Expect rough edges and potential changes to the workflow.

**When to use:** Reviewing a GitHub Pull Request for merge-readiness.

**What it does:**
1. Fetches PR metadata, diff, and base branch via `gh` CLI
2. Simulates merged state (what code looks like if PR is merged)
3. Runs all adversarial roles on merged code
4. Submits findings as GitHub PR review with inline comments

**Invocation:**
```
pr-review <PR_IDENTIFIER> [--base <BRANCH>]

Where <PR_IDENTIFIER> is one of:
  123                    PR number in current repo
  owner/repo#123         PR in specific repo  
  https://github.com/... Full PR URL

Options:
  --base <branch>        Compare against this branch (default: PR's base branch)
```

**Requirements:**
- `gh` CLI installed and authenticated (`gh auth login`)
- Repository must be a GitHub repository
- User must have write access to submit reviews

**Experimental Limitations:**
- Line mapping for inline comments may have edge cases
- Large PRs (100+ files) may take significant time
- Draft PRs supported but will submit reviews (won't affect merge status)

**Output:** GitHub PR review submitted with one of:
- `APPROVE` — No P0/P1 issues found
- `REQUEST_CHANGES` — P0/P1 issues must be fixed
- `COMMENT` — Review findings without blocking

**Workflow:**
1. Parse PR identifier (number, owner/repo#num, or URL)
2. Gather PR context (metadata, diff, commits)
3. Fetch base branch files
4. Simulate merged state for each changed file
5. Run all adversarial roles on merged files
6. Synthesize findings via Judge
7. Map findings to diff positions for inline comments
8. Submit GitHub PR review

See `references/pr-review-workflow.md` for detailed step-by-step instructions.

## GitHub Integration

### PR Context Gathering

Before PR review, gather context using `gh` CLI:

```bash
# Get PR metadata (title, body, author, branches, files)
gh pr view <number> --json title,body,author,baseRefName,headRefName,files,additions,deletions

# Get the diff
gh pr diff <number>

# Get commit history
gh api repos/:owner/:repo/pulls/:number/commits

# Fetch base branch file content
gh api repos/:owner/:repo/contents/:path?ref=:baseRef
```

### Line Mapping (CRITICAL)

GitHub PR comments use **diff positions**, not file line numbers:

- Position 1 = first line after `@@` hunk header
- Position increases through whitespace and hunks
- Position resets at each new file

**Mapping algorithm:**
1. Parse diff hunks (find `@@ -a,b +c,d @@` headers)
2. Track position counter starting at 1 for each file
3. Map findings from merged file line → diff position
4. Include in `comments[]` array with `path` and `position`

See `references/pr-line-mapping.md` for the full algorithm.

### Submitting Review

```bash
gh api repos/:owner/:repo/pulls/:number/reviews \
  -X POST \
  -f event='APPROVE|REQUEST_CHANGES|COMMENT' \
  -f body='Review summary markdown' \
  -F comments='[{"path":"file.ts","position":5,"body":"Issue description"}]'
```

See `references/pr-review-output-format.md` for the complete payload format.

## Output Format

All roles return **JSON for agent consumption**. See prompt files in `references/`:
- `references/correctness-reviewer-prompt.md` — Correctness Reviewer JSON format
- `references/challenger-prompt.md` — Challenger JSON format
- `references/reference-checker-prompt.md` — Reference Checker JSON format
- `references/security-probe-prompt.md` — Security Probe JSON format
- `references/edge-hunter-prompt.md` — Edge Hunter JSON format
- `references/simplifier-prompt.md` — Simplifier JSON format
- `references/judge-prompt.md` — Judge JSON format

**PR Review specific:**
- `references/pr-review-workflow.md` — Step-by-step PR review instructions
- `references/pr-context-gathering.md` — GitHub context extraction
- `references/pr-line-mapping.md` — Diff position mapping algorithm
- `references/pr-review-output-format.md` — GitHub API payload format

## Synthesis (CRITICAL)

**Main agent MUST synthesize before delivering to user.**

1. Parse JSON from all subagent sessions
2. Create human-readable summary using `references/synthesis-template.md`
3. Send readable summary to user (NOT raw JSON)
4. Save full JSON to `notes/code-review-TIMESTAMP.json` only if needed

See `references/synthesis-template.md` for the template.

## Model Selection

**Default:** All roles use your configured default model.

**Override:** Use a more capable model for Security Probe when deeper exploration needed:

```
task(
  subagent_type="general",
  description="Security Probe: Find vulnerabilities",
  prompt="...",
  model="anthropic/claude-sonnet-4-20250514"
)
```

Note: Model overrides require configuring agents in your opencode.json. Alternatively, invoke Security Probe via a specialized security agent if configured.

## Priority Ranking

| Priority | Meaning | Action |
|----------|---------|--------|
| **P0** | Critical | Blocks merge/deploy, security vulnerability, data loss risk |
| **P1** | Important | Should fix before merge, potential bugs, maintainability issues |
| **P2** | Minor | Future improvements, style nits, optional optimizations, simplification opportunities |
| **P3** | Nitpick | Nice to have, polish, non-blocking improvements |

## Decision Values

| Decision | Meaning |
|----------|---------|
| `APPROVE` | No P0/P1 issues, ready to merge |
| `REQUEST_CHANGES` | P0/P1 issues found, fix and re-review |
| `BLOCK` | Fundamental issues, reconsider approach |

## Role Details

### Correctness Reviewer

**Intent verification mindset:** Does this code do what it's supposed to do?

Focus areas:
- **Intent verification** — Does this solve the stated problem?
- **Input/Output verification** — Does this function return what it claims?
- **Logic correctness** — Are conditions correct (not inverted)?
- **Data flow** — Does data flow correctly through the system?
- **Test coverage** — Do tests verify correctness, not just execution?
- **Documentation alignment** — Does code match comments/documentation?

See `references/correctness-reviewer-prompt.md` for full prompt.

### Challenger

**Adversarial mindset:** Assume everything will break. Challenge every assumption.

Focus areas:
- External dependencies — "What if this library changes API?"
- Environment assumptions — "What if this runs on a different OS?"
- Configuration assumptions — "What if env vars are missing?"
- State assumptions — "What if this runs twice? In parallel?"
- Error handling — "What happens when this fails?"

See `references/challenger-prompt.md` for full prompt.

### Reference Checker

**Verification mindset:** Every reference must exist and match.

Focus areas:
- Import paths — "Does this module exist?"
- Function signatures — "Does this function accept these parameters?"
- Object methods — "Does this object have that method?"
- Type compatibility — "Is this type correct for this usage?"
- Dependency versions — "Is this API available in the current version?"

See `references/reference-checker-prompt.md` for full prompt.

### Security Probe

**Exploitation mindset:** Every input is an attack vector.

Focus areas:
- Input validation — "Can this be injected?"
- Authentication — "Can this be bypassed?"
- Authorization — "Can this escalate privileges?"
- Data exposure — "Where does sensitive data flow?"
- Secrets — "Are credentials exposed?"
- Dependencies — "Are there known vulnerabilities?"

See `references/security-probe-prompt.md` for full prompt.

### Edge Hunter

**Boundary mindset:** Test the limits.

Focus areas:
- Empty/null inputs — "What if this array is empty?"
- Concurrent access — "What if two threads call this?"
- Resource limits — "What if memory is exhausted?"
- Network failures — "What if the API times out?"
- Large inputs — "What if this list has 1M items?"
- Offline scenarios — "What if this runs without network?"

See `references/edge-hunter-prompt.md` for full prompt.

### Simplifier

**Complexity reduction mindset:** Is this as simple as possible?

Focus areas:
- **Unnecessary abstractions** — "Is this abstraction pulling its weight?"
- **Over-engineering** — "Is this solution more complex than the problem?"
- **Duplicate logic** — "Could this be consolidated?"
- **Complex conditionals** — "Could guard clauses flatten this?"
- **Dead code** — "Is this used anywhere?"
- **Naming & structure** — "Does this add clarity or confusion?"

See `references/simplifier-prompt.md` for full prompt.

**Note:** Simplifier findings are always P2/P3 (never P0/P1). Simplification never blocks merge.

### Judge

**Synthesis mindset:** Weigh findings and prioritize.

Responsibilities:
- Parse all role outputs
- Deduplicate findings
- Rank by severity (P0/P1/P2)
- Provide actionable summary
- Recommend decision (APPROVE/REQUEST_CHANGES/BLOCK)

See `references/judge-prompt.md` for full prompt.

## Code Type Detection

The skill automatically detects code type and adjusts role priority:

| Code Type | Primary Focus | Secondary Focus |
|-----------|---------------|------------------|
| **Backend API** | Correctness Reviewer, Security, Edge Hunter | Challenger, Reference Checker |
| **Frontend UI** | Correctness Reviewer, Edge Hunter, Security Probe | Challenger, Simplifier |
| **Data Pipeline** | Correctness Reviewer, Edge Hunter, Reference Checker | Challenger |
| **Infrastructure** | Security Probe, Edge Hunter, Correctness Reviewer | Reference Checker |
| **Library/SDK** | Reference Checker, Correctness Reviewer, Edge Hunter | Security Probe, Simplifier |
| **Test Code** | Challenger, Correctness Reviewer, Edge Hunter | — |

## Integration with Other Skills

| Skill | When to Use |
|-------|-------------|
| `systematic-debugging` | Bugs found during review |
| `verification-before-completion` | Pre-commit check |
| Build agent | Implementing fixes |

## Research Background

This skill is based on research from:
- HubSpot Sidekick (2026) — Judge agent pattern for filtering output
- Rasheed et al. (2025) — Multi-agent code review system
- Addy Osmani (2026) — AI code review best practices

Key insight: The judge agent is essential for filtering AI output. Without it, reviews produce "text noise" — generic observations that add no value.

## Tips

1. **Spawn in parallel** — Challenger, Reference Checker, Security Probe, Edge Hunter run simultaneously
2. **Judge runs last** — Synthesis requires all role outputs first
3. **Know when to stop** — P2s are backlog, don't perfection-cycle
4. **Code type matters** — Adjust mode based on what you're reviewing
5. **Security code needs Security Probe** — Don't skip for auth/payment code

## When NOT to Use

- **Pure style/formatting** — Use linters and formatters instead
- **Generated code review** — AI-generated code needs different approach (verify it works)
- **Massive PRs** — Break down first, then review incrementally

See `references/anti-patterns.md` for common mistakes to avoid.