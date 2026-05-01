# Code Review Skill: Research Comparison & Guardrails

## Comparison with Research Literature

### HubSpot Sidekick (March 2026)

| Aspect | HubSpot Sidekick | Our Code-Review Skill | Gap? |
|--------|------------------|----------------------|------|
| **Multi-agent approach** | Review Agent + Judge Agent | 7 roles + Judge | ✅ Covered |
| **Judge filtering** | Secondary agent filters low-value comments | Judge synthesizes all outputs | ✅ Covered |
| **Context retrieval** | RPC tools for repo context, coding conventions | Manual context provision | ⚠️ Gap |
| **Feedback quality** | 80% thumbs-up, signal-to-noise ratio | JSON output for synthesis | ✅ Covered |
| **Multi-model support** | Anthropic, OpenAI, Google fallbacks | Model selection via agentId | ✅ Covered |
| **User feedback loop** | Emoji reactions improve prompts | Missing | ⚠️ Gap |

**Key Insight from HubSpot:**
> "The most impactful change was adding a second agent to evaluate reviews before posting. The result: fewer, better, and more actionable comments."

This validates our Judge role pattern. However, HubSpot also emphasizes:
- **Context retrieval tools** (repo context, coding conventions)
- **User feedback loop** (reactions improve future reviews)

### CodeRabbit Research (February 2026)

| Finding | Implication | Our Coverage |
|---------|-------------|--------------|
| AI-generated code has **1.7x more logic/correctness bugs** | Correctness is critical | ✅ Correctness Reviewer in all modes |
| Attribution is harder than adoption | Track AI-attributed defects | ⚠️ Gap: No defect tracking |
| Multi-agent validation chains | One agent codes, another critiques, another tests | ✅ Covered (7 roles) |
| Third-party validation tools | External validation for AI agents | ⚠️ Gap: No external validation |

**Key Insight from CodeRabbit:**
> "AI-assisted code generation produces 1.7x more issues related to logical and correctness bugs."

This validates making Correctness Reviewer mandatory in all modes.

### AI Agent Guardrails (2026 Research)

| Guardrail Type | Research Recommendation | Our Coverage |
|----------------|------------------------|--------------|
| **Layer 1: Rule-based** | PII detection, format checking, blocklists | ⚠️ Gap: No rule-based validators |
| **Layer 2: ML Classifiers** | Toxicity, bias, jailbreak detection | ⚠️ Gap: No ML classifiers |
| **Layer 3: LLM Validation** | Groundedness, policy alignment | ✅ Covered (Judge) |
| **Risk-based routing** | Low/medium/high risk paths | ⚠️ Gap: Single mode selection |
| **Async validation** | Stream first, validate later | ⚠️ Gap: No async mode |

### Academic Multi-Agent Code Review (Rasheed et al., 2025)

| Agent Type | Research Model | Our Coverage |
|------------|----------------|--------------|
| **Code Review Agent** | Initial scan | ✅ Challenger |
| **Bug Report Agent** | Bug detection | ✅ Edge Hunter |
| **Code Smell Agent** | Design quality | ✅ Simplifier |
| **Code Optimization Agent** | Performance | ⚠️ Gap: No performance role |

**Note:** Research uses sequential pipeline (Review → Bug → Smell → Optimization). We use parallel execution + Judge synthesis.

### Addy Osmani Best Practices (January 2026)

| Practice | Recommendation | Our Coverage |
|----------|----------------|--------------|
| **Multi-model reviews** | Different LLMs for different purposes | ✅ Model selection |
| **AI as first-pass** | Treat AI as spellcheck, not editor | ✅ Multiple roles |
| **Human focus areas** | Architecture, security, business logic | ✅ Security Probe, Correctness |
| **45% security flaws in AI code** | Security review is critical | ✅ Security Probe |

---

## Identified Gaps

### Critical Gaps (Should Add)

1. **Context Retrieval Integration**
   - HubSpot uses RPC tools for repo context and coding conventions
   - We rely on manual context provision
   - **Recommendation:** Add context retrieval pattern to skill

2. **Performance Reviewer Role**
   - Academic research includes optimization agent
   - We don't have dedicated performance review
   - **Recommendation:** Add Performance Reviewer role

3. **User Feedback Loop**
   - HubSpot uses reactions to improve future prompts
   - We have no feedback mechanism
   - **Recommendation:** Document feedback collection pattern

### Moderate Gaps (Consider Adding)

4. **Rule-Based Pre-Filter**
   - Research recommends PII detection, format checking
   - We rely on roles for all validation
   - **Recommendation:** Add pre-filter step for obvious issues

5. **Risk-Based Mode Selection**
   - Research recommends low/medium/high risk paths
   - We have mode selection but not risk-based
   - **Recommendation:** Add risk assessment to mode selection

6. **Defect Tracking**
   - CodeRabbit emphasizes AI-attributed defect metrics
   - We don't track review outcomes
   - **Recommendation:** Add tracking recommendation

### Minor Gaps (Nice to Have)

7. **Async Validation**
   - Research recommends stream-first-validate-later for low risk
   - We're synchronous by design
   - **Low priority:** Not applicable to agent use case

8. **External Validation**
   - CodeRabbit recommends third-party validation
   - We're self-contained
   - **Low priority:** User can add external tools

---

## Recommended Guardrails

### 1. Context Retrieval Guardrail

**Problem:** Reviews without context produce generic feedback.

**Solution:** Require context before review.

```markdown
## Context Requirements

Before running code review, provide:

1. **Requirements/Spec** — What is this code supposed to do?
2. **Coding Conventions** — Style guide, patterns, anti-patterns
3. **Related Files** — Dependencies, callers, tests

**Minimum context for each mode:**

| Mode | Minimum Context |
|------|-----------------|
| `quick` | File path or snippet |
| `verify` | Requirements + file path |
| `security` | Architecture overview + file path |
| `quality` | Coding conventions + file path |
| `full` | Requirements + conventions + architecture |

**Without context, reviews will be superficial.**
```

### 2. Performance Reviewer Role

**Add to SKILL.md:**

```markdown
### Performance Reviewer

**Performance mindset:** Does this code use resources efficiently?

Focus areas:
- **Algorithmic complexity** — "Is this O(n²) when it could be O(n)?"
- **Memory usage** — "Are there memory leaks? Unnecessary allocations?"
- **I/O efficiency** — "N+1 queries? Excessive API calls?"
- **Caching** — "Could this be cached? Is caching appropriate?"
- **Concurrency** — "Race conditions? Lock contention? Deadlocks?"
- **Resource cleanup** — "Are resources properly released?"

See `references/performance-reviewer-prompt.md` for full prompt.
```

### 3. User Feedback Loop

**Add to SKILL.md:**

```markdown
## Continuous Improvement

Code review improves with feedback. After each review:

1. **Track outcomes** — Did the review find real issues?
2. **Collect reactions** — Were findings actionable?
3. **Note false positives** — What did roles get wrong?
4. **Adjust prompts** — Refine role prompts based on feedback

**Feedback collection pattern:**

```json
{
  "review_id": "timestamp-file",
  "findings_accepted": 5,
  "findings_rejected": 2,
  "false_positives": ["S3 was not actually an issue"],
  "missed_issues": ["Should have caught the null pointer"],
  "mode_used": "verify",
  "confidence": "medium"
}
```

This feedback improves future reviews and tunes role prompts.
```

### 4. Pre-Filter Layer

**Add to SKILL.md:**

```markdown
## Pre-Review Checklist

Before running roles, verify:

- [ ] Context provided (requirements, conventions)
- [ ] File type detected (backend, frontend, infrastructure)
- [ ] Language/framework identified
- [ ] Test coverage checked (does code have tests?)

**Quick checks that roles skip:**

| Check | Skip if... |
|-------|-----------|
| Linting errors | Run `eslint`/`ruff` first |
| Format issues | Run `prettier`/`black` first |
| Type errors | Run `tsc`/`mypy` first |
| Basic tests | Run test suite first |

**Roles focus on what tools can't check:**
- Intent verification
- Assumption challenges
- Security vulnerabilities
- Edge cases
- Complexity reduction
```

### 5. Risk Assessment

**Add to SKILL.md:**

```markdown
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
```

---

## Implementation Priority

| Priority | Guardrail | Effort | Impact |
|----------|-----------|--------|--------|
| **P0** | Context requirements | Low | High |
| **P0** | Risk-based mode selection | Low | High |
| **P1** | Performance Reviewer role | Medium | Medium |
| **P1** | Pre-filter checklist | Low | Medium |
| **P2** | User feedback loop | Medium | Medium |
| **P2** | Defect tracking pattern | Medium | Low |

---

## Summary

**What we got right (validated by research):**
- ✅ Multi-role approach (HubSpot, Rasheed et al.)
- ✅ Judge synthesis pattern (HubSpot)
- ✅ Correctness mandatory (CodeRabbit: 1.7x more bugs in AI code)
- ✅ Security role (Addy Osmani: 45% security flaws in AI code)
- ✅ Simplifier for code quality (Rasheed et al.)

**What we're missing (from research):**
- ⚠️ Context retrieval tools (HubSpot)
- ⚠️ Performance review (Rasheed et al.)
- ⚠️ User feedback loop (HubSpot)
- ⚠️ Risk-based routing (Authority Partners)
- ⚠️ Pre-filter layer (Authority Partners)

**Recommended immediate additions:**
1. Context requirements documentation
2. Risk-based mode selection
3. Pre-review checklist

**Recommended future additions:**
1. Performance Reviewer role
2. User feedback collection pattern
3. Defect tracking integration