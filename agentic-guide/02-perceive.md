# Perception-Gap Audit

The perception gap is the delta between what an agent can currently see and what it actually needs to make correct decisions on a task. Closing that gap before dispatching any agent is the single most reliable way to prevent wasted work, wrong assumptions, and hard-to-diagnose failures downstream.

## When to Run a Perception-Gap Audit

Run a perception-gap audit before dispatching any agent whose task involves decisions about code structure, environment configuration, external integrations, or team conventions. The audit is not optional for the Perceive phase — it is the Perceive phase. Tasks that skip it accumulate hidden assumptions that surface as bugs at verification time rather than as questions before dispatch.

## What an Agent Can Perceive

1. **File content.** Any file the agent explicitly reads or that a tool returns. The agent does not automatically read every file in the repository — only what it searches for or is directed to.
2. **Tool results.** The structured output of every tool call in the session: Bash output, file search results, MCP server responses, and test run output.
3. **Conversation history.** Every message in the current session, including previous tool calls and their outputs — the agent's working memory.
4. **System prompt.** The instructions loaded at session start: in this repository, `CLAUDE.md` and all files in `.claude/rules/`. These load automatically and define the baseline constraints.
5. **Injected context.** Content you explicitly add to the session prompt: task descriptions, scenario details, inline field lists, or pasted requirements. The primary mechanism for closing gaps the harness does not cover.

## What an Agent Cannot Perceive (Default)

The following are invisible unless you explicitly surface them. Agents that need this information silently substitute training-data assumptions, which are often wrong for your specific environment.

1. **Runtime state.** What is running in Railway, the current value of a live environment variable, or whether a migration has applied — none of this is visible unless you state it or the agent calls an MCP tool that retrieves it.
2. **Deployed configuration.** Actual Vercel or Railway environment variable values. The agent can read `.env.example` (placeholders) and code that references variables, but not their resolved values.
3. **Team conventions not in CLAUDE.md.** Decisions made verbally, in Slack, or in Notion that have not been written into `CLAUDE.md` or `.claude/rules/` are invisible — including branch-to-environment mappings and migration commands.
4. **External tickets and issue context.** GitHub issues, Jira tickets, Figma designs, and Linear tasks are not loaded automatically. Requirements in external trackers must be pasted into the session prompt.
5. **Other agents' work.** Subagents running in parallel do not share context. One agent writing a file while another edits a schema in the same area produces conflicts neither can see. Partition parallel work so each agent owns its files exclusively.

## Perception-Gap Audit Procedure

Work through these steps before writing any session prompt.

1. **List every decision the agent will make.** Be specific. For a module-creation task: which directory, which files, which shared interfaces from `packages/types`, which environment variables, which conventions?

2. **For each decision: what information does it require?** Map each decision to a concrete source. "Which directory" requires the monorepo layout. "Which shared interfaces" requires either reading `packages/types/src/` or having the interfaces defined before the agent runs.

3. **Is that information currently visible?** Check against the five perception inputs above. If the information lives in `CLAUDE.md`, a `.claude/rules/` file, or a skill, it is visible. If it lives in Slack, a Railway dashboard, or only in your head, it is not.

4. **If no: close the gap.** Four options in order of preference:
   - **Write it into CLAUDE.md or a rule file** if it is a durable constraint that applies to every future task.
   - **Create or update a skill** if it is procedural knowledge needed when scaffolding a specific pattern.
   - **Inject it into the session prompt** for task-specific context that is not reusable.
   - **Add a hook** if it is a quality gate that must fire automatically on file changes.

5. **Re-audit after the first agent run.** The first run surfaces gaps you did not anticipate. When an agent makes a wrong assumption, trace it back to the missing perception input and close it permanently.

## Context Mapping Template

Fill in one row per decision point. Any row with "Yes" in the Gap column requires action before dispatch.

| Decision                                         | Required information                               | Where it currently lives                                  | Gap? |
| ------------------------------------------------ | -------------------------------------------------- | --------------------------------------------------------- | ---- |
| Which directory to create the module in          | `apps/api/src/modules/<domain>/` convention        | `.claude/rules/nestjs-module-structure.md`                | No   |
| Which interface to implement                     | Shared DTO interface in `packages/types`           | `packages/types/src/dtos/`, `fullstack-packages-types.md` | No   |
| What DATABASE_URL resolves to                    | Railway env var for staging                        | Not in any file — lives in Railway dashboard              | Yes  |
| What Drizzle schema conventions to follow        | `$inferSelect`/`$inferInsert` usage, column naming | `.claude/rules/nestjs-dtos-drizzle.md`                    | No   |
| Whether to run a migration after schema creation | Project migration workflow and command             | Not documented in any rule or skill                       | Yes  |
| What the feature's business rules are            | Product requirements                               | Lives in Notion — not injected                            | Yes  |

## This Repository's Context Surface

Every source of context the harness loads automatically, or that you invoke manually. Use this when deciding where to close a gap.

| Source                           | Content                                                                                                         | How loaded                                         |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `CLAUDE.md`                      | Project rules, Paddle rules, Git rules, TypeScript rules, Playwright rules                                      | Automatic                                          |
| `.claude/rules/*.md`             | Auth, API client, module structure, DTO/Drizzle, deployment environments, env var conventions, coding standards | Automatic                                          |
| `.claude/skills/<name>/SKILL.md` | Scaffolds and procedures: `nestjs-create-module`, `fullstack-auth-setup`, `nextjs-api-client-setup`, others     | Invoked by skill name — not loaded until triggered |
| `.mcp.json`                      | Registered MCP servers: `paddle-sandbox`, `next-devtools`, `context7`, `playwright`, `vercel`                   | Automatic — defines callable external tools        |
| `.claude/settings.json`          | Hook definitions: Prettier formatter, TypeScript checker, audit log                                             | Automatic — hooks execute on matched events        |
| Session prompt                   | Feature requirements, scenario details, inline field lists, env var values for this run                         | Manual — you write it before dispatching           |

## Practical Example

### Scenario: Dispatching an agent to add a new NestJS module

Task: create a `tasks` feature module — entity, use case, Drizzle repository, controller, and shared DTO interface.

| Decision                                         | Required information                                                            | Where it currently lives                                    | Gap? |
| ------------------------------------------------ | ------------------------------------------------------------------------------- | ----------------------------------------------------------- | ---- |
| Folder structure                                 | `apps/api/src/modules/tasks/` with `domain/`, `application/`, `infrastructure/` | `nestjs-module-structure.md`                                | No   |
| File names and layer rules                       | Which file goes where, what each layer may import                               | `nestjs-module-structure.md` + `nestjs-create-module` skill | No   |
| Where to define `ICreateTaskDto`                 | `packages/types/src/dtos/tasks.ts` must exist first                             | `fullstack-packages-types.md`                               | No   |
| Drizzle schema placement                         | `infrastructure/persistence/tasks.schema.ts`, `$inferSelect` pattern            | `nestjs-dtos-drizzle.md`                                    | No   |
| What `DATABASE_URL` resolves to                  | `.env.local` value (git-ignored, not readable by default)                       | Not visible                                                 | Yes  |
| Whether to run a migration after schema creation | Migration command and when it runs                                              | Not documented in any rule or skill                         | Yes  |
| What fields `Task` needs                         | Product requirements                                                            | Not in any file                                             | Yes  |

**Result:** Three gaps before dispatch.

`DATABASE_URL` is not a problem for scaffold-only work — inject "do not run migration commands; scaffold files only" into the prompt.

The migration workflow gap needs a permanent fix: add `pnpm --filter api drizzle-kit generate && pnpm --filter api drizzle-kit migrate` to `.claude/rules/nestjs-dtos-drizzle.md` so every future schema task inherits it.

Entity fields are a requirement gap, not a harness gap. Supply the field list in the prompt: "A Task has `id: string`, `title: string`, `description: string | null`, `status: TaskStatus`, `userId: string`, `createdAt: Date`."

With all three gaps closed: scaffold covered by the skill, layer rules in `.claude/rules/`, DTO contract defined, and prompt scoped to files only.

## Anti-patterns

- **Dispatching without auditing.** "The agent will figure out where to put the files" is training-data guessing. If the information is not in the context surface, the agent guesses.

- **Closing gaps with corrections instead of rules.** When an agent writes a file in the wrong place, the instinct is to correct it. The right response is to write the constraint into `CLAUDE.md` or a rule file. Corrections fix one task; rules fix every future task.

- **Treating the session prompt as the only gap-closing tool.** One-off context belongs in the prompt. Context you inject into every prompt belongs in a permanent rule or skill.

- **Assuming skills load automatically.** Skills are not part of the automatic context surface. `nestjs-create-module` only fires when the agent invokes it. If your prompt lacks the trigger phrase from the skill's frontmatter, the agent proceeds without it.

- **Auditing only before the first run.** New conventions, env vars, and integrations create gaps as the project grows. Make the post-first-run re-audit a habit.

→ Next: [03-spec.md](03-spec.md)
