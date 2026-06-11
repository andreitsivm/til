# The 4-Pillar Harness

The harness is the infrastructure layer you configure once so every agent task follows the same guardrails, accesses the same knowledge, and produces consistently formatted, type-safe output. It is built from four independent but complementary primitives: Skills, Subagents, MCP servers, and Hooks.

## When to Read This Doc

Read this document when setting up the harness for the first time, when you need to understand what a specific pillar does and how it is configured here, or when adding a new skill, MCP server, or hook. If you are already familiar with the harness and want to start building a feature, skip to [02-perceive.md](02-perceive.md).

---

## Pillar 1: Skills

### What They Are

A skill is a Markdown file that gives an agent domain-specific knowledge and workflow instructions it would otherwise reconstruct on every invocation. Skills encode decisions that have already been made — which patterns to follow, which constraints apply — so agents spend their context budget executing rather than deliberating. A skill is invoked by the agent recognising a trigger condition or by the user running a slash command.

### Anatomy of a Skill File

Every skill lives at `.claude/skills/<name>/SKILL.md`. The file must begin with a YAML frontmatter block followed by the implementation content.

```markdown
---
name: skill-name
description: >
  Use when [trigger condition]. Triggers on: "[phrase1]", "[phrase2]".
  Covers [what it contains].
  Always read [related-rule.md] first.
---

# Skill Title

[Numbered procedure, code templates, or reference tables]
```

The `description` field is the most important part. A vague description means the skill gets skipped; concrete trigger phrases mean it fires reliably. Keep the implementation content actionable — numbered steps, complete code templates, and explicit constraints rather than general advice.

### Skills in This Repository

| Skill                           | Description                                                                                             |
| ------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `ai-validation`                 | Zod validation patterns for AI API responses and streaming chunks from Claude/OpenAI                    |
| `coding-standards`              | Detects code smells, anti-patterns, and readability issues during feature work and review               |
| `component-i18n`                | Ensures every user-visible string in a component routes through the translation layer                   |
| `component-theming`             | Enforces CSS variable usage for all color values so light and dark modes work automatically             |
| `documentation-criteria`        | Guides creation and review of PRDs, ADRs, Design Docs, UI specs, and Work Plans                         |
| `frontend-design`               | Theme setup, typography, color systems, and animations using shadcn/ui + Tailwind CSS                   |
| `frontend-technical-spec`       | Frontend environment variable management, component architecture, and data flow patterns                |
| `frontend-typescript-testing`   | Component testing with React Testing Library, MSW mocking, and Playwright E2E patterns                  |
| `fullstack-auth-setup`          | Auth.js session config, TypeScript module augmentation, NestJS JWT strategy, `@CurrentUser` decorator   |
| `implementation-approach`       | Selects vertical-slice, horizontal, or hybrid strategy with risk assessment before coding begins        |
| `integration-e2e-testing`       | Integration and E2E test design with mock boundaries and behavior verification rules                    |
| `nestjs-create-module`          | Full folder scaffold and code templates for a NestJS feature module in pragmatic hexagonal architecture |
| `nextjs-api-client-setup`       | Singleton Axios client configuration and BFF route templates for Next.js → NestJS calls                 |
| `nextjs-code-splitting`         | `next/dynamic` patterns, when to split vs. bundle inline, and common pitfalls to avoid                  |
| `project-context`               | Project-specific constraints, directory conventions, and external resource access methods               |
| `shadcn-dialogs`                | Typed, queued dialog management system using Zustand for Next.js + shadcn/ui projects                   |
| `skill-optimization`            | Evaluates and edits skill files using 8 content patterns and 9 editing principles                       |
| `subagents-orchestration-guide` | Task distribution, parallelism decisions, and autonomous execution mode for subagent workflows          |
| `task-analyzer`                 | Metacognitive task analysis that returns scale estimates and selects relevant skills                    |
| `technical-spec`                | Environment variable configuration, architecture design, and build/test command reference               |
| `typescript-rules`              | Enforces no-any policy, type guard patterns, and proper use of utility types in TypeScript              |
| `typescript-testing`            | Vitest test design standards, coverage requirements, and mock usage guidelines                          |

### When to Write a New Skill

1. Identify a recurring decision point where agents produce incorrect or inconsistent output — at least three separate instances before writing the skill.
2. Create `.claude/skills/<name>/SKILL.md` and write the frontmatter block first. Use the exact trigger phrases a user or agent would type.
3. Write the body as a numbered procedure with concrete commands (`run pnpm build`, `write the interface in packages/types/src/dtos/`). Include a complete code template if the skill scaffolds files.
4. Run `/skill-optimization` to evaluate against the 8 content patterns before committing.

### Anti-patterns

- **Writing a skill for a one-off task.** Skills amortize over many invocations. If the workflow will not repeat, put the instructions inline instead.
- **Vague trigger descriptions.** `Use when working on auth` is too broad. `Triggers on: "setup auth", "configure Auth.js", "add JWT strategy"` fires reliably.
- **Duplicating rule content.** Architecture constraints (`always`, `never`, `must not`) belong in `.claude/rules/`. Skills reference rules — they do not copy them.
- **Omitting a code template when the skill scaffolds something.** If the skill creates a NestJS module or an Axios client, it must contain the complete file template. Partial templates produce partial implementations.

---

## Pillar 2: Subagents

### What They Are

A subagent is a separate agent invocation with its own context window and isolated workspace. The parent dispatches it to execute a bounded unit of work and receives the result when it completes. Because subagents do not share context, they cannot carry assumptions or half-edited state from one task into another.

### Why Isolation Matters

Every agent conversation accumulates context: file contents, previous tool call outputs, in-progress reasoning. A single long-running agent handling multiple tasks carries all previous context forward — including wrong assumptions corrected mid-task. Subagents reset this accumulation: each starts clean, reads only what it needs, and returns a focused result. The parent agent's context stays bounded to coordination rather than growing with implementation details.

### Agent Types Available

The Claude Code Agent tool supports the following types:

- **Explore** — broad codebase search to answer a research question
- **Plan** — architecture or implementation plan without writing code
- **feature-dev:code-architect** — designs feature structure before implementation
- **feature-dev:code-explorer** — investigates an existing feature's current state
- **investigator** — root-cause analysis for bugs or unexpected behavior
- **solver** — implements a fix for a diagnosed problem
- **task-executor** — runs a specific, well-scoped implementation task
- **task-decomposer** — breaks a large task into parallel subtasks
- **work-planner** — produces the full sequenced plan for a multi-step feature
- **code-reviewer** — reviews a diff or file set for correctness and quality
- **quality-fixer** — applies fixes identified by a review pass
- **general-purpose** — catch-all for tasks that do not match a specific type

### When to Use a Subagent vs. Inline Work

| Situation                                        | Use a Subagent                | Do Inline |
| ------------------------------------------------ | ----------------------------- | --------- |
| Two or more tasks with no shared state           | Yes — run in parallel         | No        |
| Task requires reading 10+ files to build context | Yes — isolate the exploration | No        |
| Quick one-line change to a known file            | No                            | Yes       |
| Investigating a bug in a separate module         | Yes — investigator agent      | No        |
| Applying a fix you already understand fully      | No                            | Yes       |
| Writing tests for an already-implemented feature | Yes — frees parent context    | No        |
| Clarifying a single ambiguous requirement        | No                            | Yes       |

### Example: Dispatching a Research Subagent

```typescript
const result = await Agent({
  type: "investigator",
  prompt: `
    Investigate why the Paddle webhook handler at apps/api/src/modules/paddle/
    does not persist subscription updates when the plan changes from monthly to annual.
    Read the handler, the use case it calls, and the Drizzle repository it depends on.
    Return: the root cause, the files involved, and the minimal change needed to fix it.
    Do not write any code — return findings only.
  `,
  workdir: "/Users/mykhailoandreitsiv/projects/interactive-tasks",
});
```

The prompt bounds scope to one handler and its dependencies, asks a specific question, and specifies an output contract (findings only, no code). Bounded scope prevents context bloat; an explicit output contract prevents the subagent from drifting into implementation.

### Anti-patterns

- **Dispatching a subagent for trivial work.** The context-setup overhead is not worth it for a task you can finish in two tool calls.
- **Giving a subagent an unbounded scope.** `Investigate the whole codebase` produces a useless result. Scope to a specific module, question, or file set.
- **Letting subagents share mutable state.** Two agents writing to the same file in parallel produces conflicts. Partition so each subagent owns its files exclusively.
- **Skipping the output contract.** Without an explicit format, subagents return whatever they consider relevant. State whether you want findings only, a diff, file paths, or a full implementation.

---

## Pillar 3: MCP (Model Context Protocol)

### What It Is

MCP (Model Context Protocol) is a standard interface that lets agents call external tools — APIs, documentation servers, browser automation engines, deployment platforms — as if they were built-in capabilities. Each server exposes named tools; the agent calls them and receives structured results. Because servers are explicitly registered in `.mcp.json`, the tool surface is bounded and auditable: an agent cannot reach a service that is not registered.

### MCP Servers in This Repository

| Server           | Purpose                                                                                                                                                                 |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `paddle-sandbox` | Paddle billing API in sandbox mode — create products, prices, subscriptions, and customers without touching live data                                                   |
| `next-devtools`  | Next.js-specific tooling: current docs, dev server interaction, version-specific API reference for the breaking-change version in use                                   |
| `context7`       | Fetch current library documentation for any framework or SDK — prevents agents from relying on stale training data for React, Drizzle, Auth.js, or any other dependency |
| `playwright`     | Browser automation — navigate, click, fill forms, take screenshots, capture network requests; used to visually validate UI changes before commit                        |
| `vercel`         | Vercel deployment management — query preview deployments, manage environment variables, check build and deployment status                                               |

### When to Add a New MCP Server

1. Confirm no existing server covers the capability. Check `.mcp.json` and the table above before proceeding.
2. Find the official MCP server for the service. Prefer servers maintained by the provider over community wrappers.
3. Add the entry to `.mcp.json` with the correct `command`, `args`, and `env` keys. Reference secrets via environment variables — never inline them.
4. Invoke one read-only tool from the new server before writing code that depends on it. A server that fails to start silently breaks every agent task that reaches for it.

### Anti-patterns

- **Calling a live server when a sandbox exists.** This repository uses `paddle-sandbox` for all development. `paddle-live` is for explicit production operations only. See `CLAUDE.md`.
- **Adding a server for a one-time lookup.** Use `context7` for ad-hoc doc fetches. Register a server only for permanent, recurring integrations.
- **Inlining credentials in `.mcp.json`.** All secrets must reference environment variables — `.mcp.json` is committed to the repository.
- **Registering a server without testing it.** An unreachable server causes silent failures: the agent calls the tool, receives an error, and may hallucinate a result instead of surfacing the failure.

---

## Pillar 4: Hooks

### What They Are

Hooks are shell commands the harness executes automatically when specific events occur, independent of agent intent. The agent does not need to remember to run Prettier or check TypeScript — the hook fires on every matching event, unconditionally.

### Event Types

| Event          | When It Fires                              | Use For                                           |
| -------------- | ------------------------------------------ | ------------------------------------------------- |
| `PostToolUse`  | After any registered tool call completes   | Formatting, linting, type-checking, audit logging |
| `PreToolUse`   | Before a registered tool call executes     | Validation, permission checks, dry-run previews   |
| `Notification` | When the agent sends a message to the user | Alerting, logging communication events            |
| `Stop`         | When the agent completes its turn          | Post-run summaries, cleanup, status reporting     |

### Hooks in This Repository

| Event         | Matcher                  | Command                             | What It Enforces                                                                                                        |
| ------------- | ------------------------ | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `PostToolUse` | `Write\|Edit\|MultiEdit` | `npx --yes prettier --write <file>` | Auto-formats every file immediately after it is written or edited                                                       |
| `PostToolUse` | `Write\|Edit\|MultiEdit` | `node $PWD/.claude/hooks/tsc.js`    | Runs TypeScript type-checking after every file change; surfaces type errors before the agent moves to the next step     |
| `PostToolUse` | `*` (all tools)          | `jq . > post-log.json`              | Writes the full tool call payload to `post-log.json` for debugging and audit; fires on every tool use without exception |

The first two hooks work in sequence: Prettier formats the file, then the TypeScript checker validates it. A type error is therefore visible on the very next tool call, before the agent edits further files downstream. The audit log hook is silent — no output to the agent's context — but records every tool invocation for post-session debugging.

### When to Add a New Hook

1. Identify a quality gate that must fire unconditionally — something the agent must never skip. If the check is advisory, put it in a skill instead.
2. Choose the narrowest event type and matcher: use `Write|Edit|MultiEdit` rather than `*` when the hook only applies to file changes.
3. Add the entry to `.claude/settings.json`. The command receives the tool payload via stdin as JSON; use `jq` to extract the file path or other fields.
4. Trigger one affected tool call and verify the side-effect occurred. Check that the command exits cleanly on both success and error paths.

### Anti-patterns

- **Using a hook for something that requires judgment.** Hooks are for unconditional side-effects. If the action needs conditional logic, encode it in a skill.
- **Using `*` when you only need file-change events.** A broad matcher fires on reads as well as writes, adding latency and log noise.
- **Calling external services from a hook.** Hooks fire on every matched event. An HTTP call to a billing API on every file write causes rate-limit failures and unexpected side-effects.
- **Skipping the exit-code test.** A non-zero exit code is a hook failure. Append `|| true` only after confirming the failure case should be swallowed rather than surfaced.

---

## The 4 Pillars Working Together

No pillar is effective in isolation. A `fullstack-auth-setup` skill that prescribes webhook verification is useless if `paddle-sandbox` is not registered in `.mcp.json`. Subagents isolate each step in a plan, but they need skills to produce correct implementations and hooks to enforce formatting and type safety on their output. Hooks fire unconditionally on file changes, but cannot substitute for the procedural knowledge in skills or the parallelism subagents provide. Configure all four and the system becomes self-correcting: agents follow established patterns, work is isolated so mistakes do not compound, external integrations are bounded and auditable, and every file change is automatically validated before the next step begins.

---

→ Next: [02-perceive.md](02-perceive.md)
