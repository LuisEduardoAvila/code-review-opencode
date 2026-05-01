# Code Review Anti-Patterns

Common mistakes to avoid when using adversarial code review.

## Anti-Pattern 1: Using for Style/Formatting

**Wrong:**
```
code-review mode: quick
[Code with style issues like indentation, naming]
```

**Why:** Linters and formatters handle this faster and more consistently.

**Right:** Use linters (eslint, ruff, prettier) for style. Use code-review for:
- Security vulnerabilities
- Logic bugs
- Edge cases
- Architecture issues

---

## Anti-Pattern 2: Reviewing AI-Generated Code the Same Way

**Wrong:**
```
code-review mode: full
[AI-generated code]
```

**Why:** AI-generated code has different failure modes — it often "looks right" but has subtle logic errors.

**Right:** For AI-generated code:
1. **Verify it works first** — Run tests, manual testing
2. **Use `verify` mode** — Focus on Reference Checker
3. **Check common AI mistakes:**
   - Hallucinated imports
   - Non-existent methods
   - Plausible but wrong logic
   - Missing error handling

---

## Anti-Pattern 3: Reviewing Massive PRs

**Wrong:**
```
code-review mode: full
[5000 line PR with 50 files]
```

**Why:** Adversarial review is thorough — it will produce too many findings, overwhelming developers.

**Right:**
1. **Break down the PR** — Smaller, focused changes
2. **Review incrementally** — Focus on critical files first
3. **Use `quick` mode** — For large PRs, quick sanity check only
4. **Request architectural review** — For design concerns, not line-by-line

---

## Anti-Pattern 4: Skipping Security Review for "Simple" Code

**Wrong:**
```
[Code with auth/payment logic]
code-review mode: quick
```

**Why:** Auth and payment code ALWAYS needs security review, no matter how "simple".

**Right:**
1. **Always use `security` mode for:**
   - Authentication code
   - Authorization checks
   - Payment processing
   - Data handling (PII, secrets)
   - API endpoints
2. **Use `full` mode for critical paths**

---

## Anti-Pattern 5: Trusting Without Verification

**Wrong:**
```
[Challenger output says "no issues found"]
→ APPROVE
```

**Why:** One role may miss issues. Adversarial review needs multiple perspectives.

**Right:**
1. **Always run multiple roles** — At least Challenger + Reference Checker
2. **Let Judge synthesize** — Don't skip the synthesis step
3. **Read the findings** — Don't just check the decision

---

## Anti-Pattern 6: Perfection-Cycling on P2s

**Wrong:**
```
[Code with P2 issues only]
→ Fix P2s
→ Review again
→ Fix more P2s
→ Review again
```

**Why:** P2s are backlog items — they're minor improvements, not blockers.

**Right:**
1. **P0/P1 → Fix immediately**
2. **P2 → Document and move on**
3. **Only re-review for P0/P1 fixes**

---

## Anti-Pattern 7: Using Wrong Model for Security

**Wrong:**
```bash
sessions_spawn(task="Security Probe: ...", model="openrouter/free")
```

**Why:** Security review needs thorough exploration. Free/small models may miss vulnerabilities.

**Right:**
```bash
sessions_spawn(task="Security Probe: ...", model="ollama/kimi-k2.5:cloud")
# Or use default glm-5:cloud
```

---

## Anti-Pattern 8: Reviewing Without Context

**Wrong:**
```
[Code snippet without surrounding context]
code-review mode: full
```

**Why:** Adversarial review needs to understand how code connects to the rest of the system.

**Right:**
1. **Provide file paths** — Not just snippets
2. **Include imports** — Reference Checker needs them
3. **Show usage** — How is this function called?
4. **Describe architecture** — What system is this part of?

---

## Anti-Pattern 9: Ignoring Verified Good Findings

**Wrong:**
```
[Judge output with "verified_good" section]
→ Focus only on issues
```

**Why:** The verified_good section shows what's working — important for confidence and avoiding false positives.

**Right:**
1. **Include verified_good in summary**
2. **Acknowledge what's working**
3. **Use for context** — "All imports correct BUT method doesn't exist"

---

## Anti-Pattern 10: Blocking on Every Finding

**Wrong:**
```
[Code with P2 style suggestions]
→ BLOCK
```

**Why:** Not all findings are blockers. The decision should reflect severity.

**Right:**
- **P0** → REQUEST_CHANGES or BLOCK
- **P1** → REQUEST_CHANGES (usually)
- **P2 only** → APPROVE (document P2s)

---

## Summary Table

| Anti-Pattern | Wrong Approach | Right Approach |
|--------------|----------------|-----------------|
| Style review | code-review for formatting | Use linters/formatters |
| AI-generated code | Full review same as human | Verify first, check AI mistakes |
| Massive PRs | Full adversarial review | Break down, use quick mode |
| Auth/payment code | Quick mode | Always use security mode |
| Trusting one role | Skip synthesis | Multiple roles + Judge |
| P2 perfection | Fix all P2s immediately | Document P2s, move on |
| Wrong model | Free model for security | Use capable model |
| No context | Snippet only | Provide file paths, context |
| Ignoring verified_good | Focus only on issues | Include what's working |
| Blocking all | BLOCK on everything | Severity-based decision |