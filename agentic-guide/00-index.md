# AI Agentic Engineering Reference Guide

AI Agentic Engineering is the discipline of designing, configuring, and operating AI agent systems so they reliably deliver production-quality software — not just single completions, but multi-step workflows with context, guardrails, and automated verification. This guide covers the four infrastructure pillars you configure once and the five-phase workflow your agents follow on every task.

---

## The 4-Pillar Framework

Your harness is built from four configurable primitives. Each one answers a different question about how your agents should behave.

| Pillar          | Purpose                                                                                                  | Configured in                                            |
| --------------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **Skills**      | Domain knowledge and workflow templates agents invoke at decision points                                 | `.claude/skills/<name>/SKILL.md`                         |
| **Subagents**   | Isolated execution contexts that run tasks in parallel without shared state                              | Dispatched via `superpowers:dispatching-parallel-agents` |
| **MCP Servers** | External tool integrations that extend what agents can read and write                                    | `.mcp.json`                                              |
| **Hooks**       | Automated side-effects (format, lint, type-check) that fire on file changes, independent of agent intent | `.claude/settings.json`                                  |

The four pillars work as a system, not in isolation. Skills tell agents **what to do** in a given situation — which patterns to follow, which checks to run, which constraints apply. Subagents isolate **who does it** — each agent gets its own context window and cannot accidentally bleed state from one task into another. MCP servers control **what they can touch** — agents can only call tools that are explicitly registered, so the blast radius of any mistake is bounded. Hooks enforce **what happens automatically** — post-write formatting, type-checking, and audit logging fire on every file change regardless of whether the agent remembered to trigger them. Invest in all four and your system becomes both capable and predictable.

---

## The 5-Phase Workflow

Every non-trivial task moves through five phases in sequence. Skipping a phase is the most common cause of rework.

```
Perceive → Spec → Plan → Dispatch → Verify
```

| Phase        | Central Question                                                            | Guide                            |
| ------------ | --------------------------------------------------------------------------- | -------------------------------- |
| **Perceive** | What is the current state of the codebase, and what is the agent missing?   | [02-perceive.md](02-perceive.md) |
| **Spec**     | What exactly should be built, and what are the acceptance criteria?         | [03-spec.md](03-spec.md)         |
| **Plan**     | What is the ordered sequence of steps, and which can run in parallel?       | [04-plan.md](04-plan.md)         |
| **Dispatch** | Which agents run which steps, and how are they coordinated?                 | [05-dispatch.md](05-dispatch.md) |
| **Verify**   | Does the output meet every acceptance criterion and pass all quality gates? | [06-verify.md](06-verify.md)     |

The flow is linear by design. Perception gaps that are not resolved before spec writing produce specs with wrong assumptions. Plans built on incomplete specs dispatch agents toward the wrong goals. Skipping verification means you discover failures in production rather than in the pipeline. The discipline is in doing the phases in order, every time.

---

## How to Navigate This Guide

Find your current situation in the table below and go directly to the relevant guide.

| Situation                                        | Go to                                |
| ------------------------------------------------ | ------------------------------------ |
| First time setting up the harness                | [01-harness.md](01-harness.md)       |
| Starting a new feature                           | [03-spec.md](03-spec.md)             |
| Need to run multiple agents in parallel          | [05-dispatch.md](05-dispatch.md)     |
| Tests are failing or CI is blocked               | [06-verify.md](06-verify.md)         |
| AI costs are too high                            | [07-cost.md](07-cost.md)             |
| Rolling out the harness to a team                | [08-production.md](08-production.md) |
| Understanding any single pillar in depth         | [01-harness.md](01-harness.md)       |
| Auditing what your agents can currently perceive | [02-perceive.md](02-perceive.md)     |

---

## Course Map

This guide is the companion reference for the four-week Neoversity curriculum on AI Agentic Engineering. Each week maps to specific guide files.

| Week   | Topic                                           | Guide Files                                                      |
| ------ | ----------------------------------------------- | ---------------------------------------------------------------- |
| Week 1 | Harness Setup and Agent Perception              | [01-harness.md](01-harness.md), [02-perceive.md](02-perceive.md) |
| Week 2 | Agent Topology and MCP Integration              | [04-plan.md](04-plan.md), [05-dispatch.md](05-dispatch.md)       |
| Week 3 | Spec-Driven Development and Quality Gates       | [03-spec.md](03-spec.md), [06-verify.md](06-verify.md)           |
| Week 4 | Scale, Cost Engineering, and Production Rollout | [07-cost.md](07-cost.md), [08-production.md](08-production.md)   |

---

→ Next: [01-harness.md](01-harness.md)
