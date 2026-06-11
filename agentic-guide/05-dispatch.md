# Multi-Agent Orchestration

Multi-agent orchestration is the practice of splitting a single feature's implementation into named workstreams, each assigned to an isolated agent that owns a bounded set of files and returns a verifiable result. The orchestrator — an agent or human — coordinates the order of dispatch, collects results, and merges outputs, but never performs the implementation work.

## When to Dispatch Multiple Agents

Dispatch multiple agents when:

- The plan contains two or more parallel task groups with no shared file ownership (see [04-plan.md](04-plan.md))
- The feature touches two or more independent stack layers (`packages/types`, `apps/api`, `apps/web`) with no immediate import dependency between the parallel tasks
- Each workstream is large enough to justify coordination overhead — a 5-minute task is not worth an agent boundary

Work inline (single agent) when:

- The task is a single-file change with one verification command
- Every step imports the previous step's output — no parallelism is available
- You are debugging — iterative inspection of shared state does not parallelize well

## The Two Topologies

### Hub-and-Spoke

In a Hub-and-Spoke topology, one orchestrator agent is responsible for reading the spec, building the dispatch plan, spawning worker agents, and collecting their results. Worker agents are stateless with respect to each other — they receive a prompt, execute a bounded task, and return. They never communicate directly with one another.

**When to use:** Any feature with a clear root dependency (usually `packages/types` interfaces) that fans out into independent backend, frontend, and test workstreams after that root is satisfied. Hub-and-Spoke is the default topology for this stack.

The orchestrator reads the spec, confirms the File Map has no overlapping file ownership, dispatches the root agent and waits for it to complete, then fans out to the remaining agents and collects their return reports.

Worker agents own only the files in their dispatch prompt, read (but never modify) earlier agents' outputs, run the specified verification command, and report: files modified, verification result, any blocker.

**Example dispatch sequence (tasks feature):**

```
Orchestrator reads spec → dispatches Agent A (types)
Agent A completes → dispatches Agent B (backend) + Agent C (API layer) in sequence
                 → also dispatches Agent D (frontend) + Agent E (quality) in parallel
```

Agent B and C are sequential because Agent C's infrastructure code imports domain types that Agent B creates. Agent D and E start immediately after A — they depend only on `packages/types` interfaces, not on the NestJS implementation.

### Peer Mesh

In a Peer Mesh topology, agents are dispatched simultaneously with no central orchestrator managing their order. Each agent is given the full spec and is responsible for detecting when its own dependencies are satisfied before proceeding.

**When to use:** Only for workstreams that are truly independent from the first line of code — parallel documentation, parallel infrastructure configuration, or scaffolding of entirely separate modules with no shared interfaces.

**Requires worktree isolation.** Every Peer Mesh agent must work in its own git worktree; concurrent writes to the same branch cause conflicts. See the Git Worktree Fan-Out section.

Peer Mesh is uncommon on this stack because most features share the `packages/types` layer, which creates a mandatory sequential root. Default to Hub-and-Spoke.

## The 5-Agent Pattern

The 5-Agent Pattern is the standard dispatch configuration for a full-stack feature on this monorepo. Five agents map to five distinct responsibilities with exclusive file ownership and a two-phase dispatch sequence.

### Splitting a Feature Into 5 Workstreams

| Agent             | Workstream                                                                        | File scope                                                                                               |
| ----------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| A: Types          | Shared TypeScript interfaces                                                      | `packages/types/src/dtos/<domain>.ts`, `packages/types/src/dtos/index.ts`, `packages/types/src/index.ts` |
| B: Backend domain | Domain entity, repository interface, and use cases                                | `apps/api/src/modules/<domain>/domain/` and `apps/api/src/modules/<domain>/application/`                 |
| C: API layer      | Drizzle schema, repository implementation, controller, DTO classes, NestJS module | `apps/api/src/modules/<domain>/infrastructure/` and `apps/api/src/modules/<domain>/<domain>.module.ts`   |
| D: Frontend       | Next.js page component and BFF proxy route                                        | `apps/web/app/[locale]/<domain>/` and `apps/web/app/api/<domain>/`                                       |
| E: Quality        | Unit tests for use cases and type checks                                          | `apps/api/src/modules/<domain>/application/use-cases/*.spec.ts`                                          |

### Example Dispatch Matrix

The following matrix uses the hypothetical `tasks` feature as a concrete example, with real file paths from this repository.

| Agent        | Task                                                                                            | Files touched                                                                                                                                                                                                                                                                                                                                                                                                                      | Tools needed                    | Isolation                                                                        |
| ------------ | ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | -------------------------------------------------------------------------------- |
| A: Types     | Add `ICreateTaskDto`, `ITaskResponse`, `IUpdateTaskDto`                                         | `packages/types/src/dtos/tasks.ts` (create), `packages/types/src/dtos/index.ts` (modify), `packages/types/src/index.ts` (modify)                                                                                                                                                                                                                                                                                                   | Read, Edit, Bash (`pnpm build`) | None — types package is the shared root; no other agent runs concurrently with A |
| B: Backend   | Task entity + repository interface + `CreateTask` and `GetTasks` use cases                      | `apps/api/src/modules/tasks/domain/task.entity.ts`, `apps/api/src/modules/tasks/domain/task.repository.ts`, `apps/api/src/modules/tasks/application/use-cases/create-task.use-case.ts`, `apps/api/src/modules/tasks/application/use-cases/get-tasks.use-case.ts`                                                                                                                                                                   | Read, Edit                      | Worktree                                                                         |
| C: API layer | Drizzle schema + Drizzle repository implementation + TaskController + DTO classes + TasksModule | `apps/api/src/modules/tasks/infrastructure/persistence/task.schema.ts`, `apps/api/src/modules/tasks/infrastructure/persistence/drizzle-task.repository.ts`, `apps/api/src/modules/tasks/infrastructure/http/tasks.controller.ts`, `apps/api/src/modules/tasks/infrastructure/http/dto/create-task.dto.ts`, `apps/api/src/modules/tasks/infrastructure/http/dto/task-response.dto.ts`, `apps/api/src/modules/tasks/tasks.module.ts` | Read, Edit, Bash (`pnpm lint`)  | Worktree (sequential after B)                                                    |
| D: Frontend  | Task list page component + BFF proxy route                                                      | `apps/web/app/[locale]/tasks/page.tsx` (create), `apps/web/app/api/tasks/route.ts` (create)                                                                                                                                                                                                                                                                                                                                        | Read, Edit                      | Worktree                                                                         |
| E: Quality   | Unit tests for use cases + type checks                                                          | `apps/api/src/modules/tasks/application/use-cases/create-task.use-case.spec.ts` (create), `apps/api/src/modules/tasks/application/use-cases/get-tasks.use-case.spec.ts` (create)                                                                                                                                                                                                                                                   | Read, Edit, Bash (`pnpm test`)  | Worktree                                                                         |

### Sequential Constraint

Dispatch follows a two-phase schedule:

**Phase 1 — Root:** Dispatch Agent A alone. Agent A creates `packages/types/src/dtos/tasks.ts` and updates the barrel exports. Its verification command (`pnpm build` in the types package) must pass before Phase 2 begins.

**Phase 2 — Parallel fan-out:** After Agent A completes, dispatch B, C, D, and E. Agent D and Agent E start in parallel — they depend only on `packages/types` interfaces. Agent B must complete before Agent C starts, because Agent C imports domain types (entity, repository interface) that Agent B creates.

```
Phase 1:   [A: Types] ──────────────────────────── complete
                                                        │
Phase 2:   ┌── [B: Backend domain] → [C: API layer]    │
           │                                            │ (all start after A)
           ├── [D: Frontend]                            │
           │                                            │
           └── [E: Quality]  ───────────────────────── │
```

The feature is done when all Phase 2 agents report completion with passing verification commands.

## Git Worktree Fan-Out

### Why Worktrees

When multiple agents write to the same working tree simultaneously, the last write wins — or writes are serialized in an order neither agent predicted. Git worktrees give each agent an isolated working directory that shares the same object store but has its own HEAD and index. Agents commit independently and the orchestrator merges at the end. A worktree is not a full clone: it reuses the existing object database, so checkout is fast and disk overhead is one extra copy of working files.

### Commands

Create a worktree for each parallel agent before dispatching. The following commands use the actual repository name `interactive-tasks`:

```bash
# From the repository root — create one worktree per parallel agent
git worktree add ../interactive-tasks-agent-b feature/tasks-backend
git worktree add ../interactive-tasks-agent-c feature/tasks-api-layer
git worktree add ../interactive-tasks-agent-d feature/tasks-frontend
git worktree add ../interactive-tasks-agent-e feature/tasks-quality

# List active worktrees
git worktree list

# After all agents complete and their branches are merged back into dev:
git worktree remove ../interactive-tasks-agent-b
git worktree remove ../interactive-tasks-agent-c
git worktree remove ../interactive-tasks-agent-d
git worktree remove ../interactive-tasks-agent-e
```

Each worktree path becomes the working directory passed to the spawned agent in its dispatch prompt. Agent A does not require a worktree — it is the only agent in Phase 1. Phase 2 worktrees are created after Agent A commits, so each one branches from a state that already includes the shared types.

### When to Use Worktrees vs. Skip Them

| Scenario                                                | Use worktrees? | Reason                                                             |
| ------------------------------------------------------- | -------------- | ------------------------------------------------------------------ |
| Two or more agents run concurrently on the same feature | Yes            | Prevents conflicts and silent overwrites                           |
| Strictly sequential agents (one runs, then the next)    | No             | No concurrent writes; sequential agents can share the working tree |
| Single-agent task                                       | No             | Nothing to isolate                                                 |
| Peer Mesh topology                                      | Always         | Peer Mesh is undefined without worktrees                           |
| Hub-and-Spoke Phase 1 (root agent only)                 | No             | Only one agent active                                              |
| Hub-and-Spoke Phase 2 (fan-out agents)                  | Yes            | Multiple agents active simultaneously                              |
| Agent that only reads files (never writes)              | No             | Read-only access cannot conflict                                   |

## Writing Agent Prompts for Dispatch

### The Prompt Completeness Rule

An agent can only use information present in its prompt or readable from its assigned files. If a constraint is not in the prompt, the agent will not apply it. If a file path is not listed, the agent may invent one. If the verification command is missing, the agent may skip it. Write prompts as if the agent has never seen this codebase.

### 5 Required Elements in Every Dispatch Prompt

Every dispatch prompt must contain all five of the following elements. A prompt missing any one of them is incomplete:

1. **Task description** — one or two sentences stating the exact outcome required, using the language of the spec's acceptance criteria
2. **Assigned files** — the complete list of files the agent may create or modify, with absolute paths; files not on this list must not be touched
3. **Dependency files to read** — files produced by earlier agents that this agent needs to understand but must not modify (e.g., interfaces from `packages/types`)
4. **Spec reference** — the acceptance criteria or spec section this task must satisfy, quoted or linked directly
5. **Verification command** — the exact shell command that must pass before the agent reports completion, and what output indicates success

### Full Prompt Template for a Hub-and-Spoke Worker Agent

Prompt for Agent B (backend domain), dispatched after Agent A has completed:

---

```
You are Agent B: Backend Domain. You are implementing the domain layer for the tasks feature.

TASK
Create the Task domain entity, the repository interface, and two use cases: CreateTask and GetTasks.
These components must satisfy AC-2 and AC-3 from the tasks feature spec.

ASSIGNED FILES (you may create or modify only these)
- apps/api/src/modules/tasks/domain/task.entity.ts (create)
- apps/api/src/modules/tasks/domain/task.repository.ts (create)
- apps/api/src/modules/tasks/application/use-cases/create-task.use-case.ts (create)
- apps/api/src/modules/tasks/application/use-cases/get-tasks.use-case.ts (create)

DEPENDENCY FILES (read only — do not modify)
- packages/types/src/dtos/tasks.ts — ICreateTaskDto, ITaskResponse, IUpdateTaskDto interfaces that your use cases must align with
- packages/types/src/auth.ts — IJwtPayload, used to type the authenticated user parameter
- .claude/rules/nestjs-module-structure.md — the architectural conventions this layer must follow
- .claude/rules/nestjs-dtos-drizzle.md — DTO and entity conventions

SPEC REFERENCE
AC-2: A POST /tasks endpoint accepts { title: string; description?: string } and persists a new task owned by the authenticated user.
AC-3: A GET /tasks endpoint returns all tasks owned by the authenticated user, ordered by createdAt descending.

CONSTRAINTS
- The domain layer (domain/ and application/) must have zero NestJS, Drizzle, or class-validator imports
- Use cases must depend on the repository interface (ITaskRepository), not any concrete implementation
- The Task entity must carry its own ID generation (use crypto.randomUUID()) — do not rely on the database for ID

VERIFICATION
Run the following command. It must exit with code 0 before you report completion:
  cd apps/api && pnpm check-types

RETURN FORMAT
Report back with:
1. List of files created or modified
2. The exact output of the verification command
3. Any blocking issue that required a deviation from the assigned files list (if none, say "none")
```

---

The prompt gives Agent B exactly what it needs. It cannot accidentally modify the types package or the infrastructure layer because those files are absent from its assigned list, and it cannot skip type-checking because the verification command is explicit.

## Token Budget Per Agent

The table below shows [course benchmark] figures for the tasks feature. Use these as a [suggested operational target] when estimating costs before dispatch.

| Agent                | Typical input tokens | Why                                                                                                                   |
| -------------------- | -------------------- | --------------------------------------------------------------------------------------------------------------------- |
| A: Types             | ~3,000               | Small scope: two or three interface files plus existing `packages/types` barrel structure                             |
| B: Backend domain    | ~5,000               | Reads rule files (nestjs-module-structure, nestjs-dtos-drizzle), the types interfaces from A, and writes four files   |
| C: API layer         | ~7,000               | Reads domain files from B, reads Drizzle schema conventions, writes six files including controller and module wiring  |
| D: Frontend          | ~6,000               | Reads Next.js API client rules, reads types interfaces, writes page component and BFF route                           |
| E: Quality           | ~7,000               | Reads all domain and use case files it is testing, writes spec files, runs test suite                                 |
| **Total (5 agents)** | **~28,000**          | [course benchmark] vs ~40,000 [suggested operational target] for a single sequential agent doing all five workstreams |

The ~28k benchmark assumes agents are given focused prompts with only the required context. Agents given the full CLAUDE.md, all rule files, and an unfiltered codebase snapshot will consume significantly more tokens without producing better output. Tight prompts are the primary lever for cost control in multi-agent dispatch.

## Anti-patterns

- **Dispatching before the types agent completes.** Agents B through E all import from `packages/types`. If those interfaces do not exist yet, the agents either invent their own local types (causing drift) or fail with import errors. Always treat Agent A's completion as a hard gate before Phase 2.

- **Giving every agent the full CLAUDE.md and all rule files.** Include only the rule files that apply to each agent's assigned files. Agent C does not need the Paddle integration rules; Agent D does not need the NestJS hexagonal architecture rules. Irrelevant context pushes load-bearing instructions down in the prompt, increasing the chance they are ignored.

- **Listing the same file in two agents' assigned lists.** When two agents write to the same file concurrently, the result is unpredictable. The File Map from the plan phase (see [04-plan.md](04-plan.md)) prevents this — every file must appear in exactly one agent's assigned list. If you find an overlap, resolve it before dispatch, not after.

- **Omitting the verification command from the dispatch prompt.** Agents that are not told how to verify their work will either skip verification or invent a step that does not match the actual quality gates. The command in the dispatch prompt must be the exact command CI runs.

- **Using Peer Mesh without worktrees.** Running two agents concurrently on the same working directory is a race condition, not a topology. If your approach requires simultaneous agents, create worktrees first. `git worktree add` before dispatch; `git worktree remove` after merge.

- **Dispatching an agent with no return format specified.** Without a structured return format, agents produce narrative summaries that are difficult to evaluate. Specify the exact fields you need back (files modified, verification output, blockers) so the orchestrator can confirm completion without reading prose.

- **Re-dispatching a failed agent without diagnosing the failure.** When an agent returns a non-zero exit code, read the error output first. Most failures are: missing dependency (types agent did not complete cleanly), constraint violation (agent modified a file outside its list), or environment issue (missing package or migration). Diagnose before re-dispatch.

→ Next: [06-verify.md](06-verify.md)
