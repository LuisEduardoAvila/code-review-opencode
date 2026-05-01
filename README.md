# Code Review - OpenCode Skill

Multi-perspective adversarial code review with specialized roles for OpenCode.

## Overview

Unlike traditional code review that finds bugs, adversarial code review **actively tries to break the code**. Each role attacks from a different angle — assumptions, references, security, edge cases.

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

## Usage

See `SKILL.md` for full documentation.

## License

MIT