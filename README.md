# Code Review - OpenCode Skill

Multi-perspective adversarial code review with specialized roles for OpenCode.

## Overview

Unlike traditional code review that finds bugs, adversarial code review **actively tries to break the code**. Each role attacks from a different angle — assumptions, references, security, edge cases.

> ⚠️ **PR Review Mode is experimental.** Expect rough edges and potential workflow changes. See `SKILL.md` for details.

## Roles

| Role | Focus | Approach |
|------|-------|----------|
| **Correctness Reviewer** | Intent verification | "Does this code do what it's supposed to do?" |
| **Challenger** | Attack assumptions | "What if this API changes? What's the blast radius?" |
| **Reference Checker** | Verify existence | "Does this function exist? Is this import path correct?" |
| **Security Probe** | Exploit vectors | "Can this input be injected? Where does data flow?" |
| **Edge Hunter** | Boundary cases | "Empty arrays? Null objects? Concurrent access?" |
| **Simplifier** | Reduce complexity | "Is this abstraction necessary? Could this be simpler?" |
| **Judge** | Synthesis | Weigh severity, prioritize fixes |

## Modes

| Mode | Roles | Use Case |
|------|-------|----------|
| `quick` | Correctness + Challenger | Pre-commit sanity check |
| `verify` | Correctness + Challenger + Reference Checker | CI/CD gate (default) |
| `security` | Correctness + Challenger + Security Probe | Before deploy |
| `quality` | Correctness + Simplifier | Technical debt review |
| `full` | All seven roles | PR review |
| `pr-review` | All roles + GitHub integration | **GitHub PR review (experimental)** |

## Usage

See `SKILL.md` for full documentation.

### PR Review Mode (Experimental)

Review GitHub PRs with merged state simulation:

```bash
pr-review 123                        # PR #123 in current repo
pr-review owner/repo#123             # Specific repo
pr-review https://github.com/...     # Full URL
pr-review 123 --base main            # Compare against specific branch
```

**Requirements:** `gh` CLI installed and authenticated.

## Comparison to Industry Tools

| Approach | This Skill | CodeRabbit | GitHub Copilot |
|----------|------------|------------|----------------|
| Mindset | Adversarial (attack) | Find bugs | Find bugs |
| Agents | 7 roles + Judge | Single agent | Single model |
| State | Merged simulation | Diff review | Context window |
| Output | GitHub review API | PR comments | PR review |

See `references/research-comparison.md` for detailed analysis.

## License

MIT