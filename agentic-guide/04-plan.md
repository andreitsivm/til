# Implementation Planning

A plan is a sequenced list of tasks derived from a spec's acceptance criteria, where each task is independently committable, independently verifiable, and narrow enough to complete in a single agent session. Plans transform a written contract (the spec) into an execution schedule that agents and human reviewers can track without ambiguity.

## When to Write a Plan

Write a plan whenever:

- The spec has more than three acceptance criteria
- Work touches more than one layer of the stack (e.g., `packages/types` plus NestJS plus Next.js)
- Multiple agents or sessions will work in parallel on the same feature
- The feature has a non-trivial dependency chain that determines execution order

Skip the plan for single-file tasks with one clear outcome — a plan for "add `updatedAt` field to `ITaskResponse`" is overhead, not value. The signal is whether a second person could execute the task from the spec alone without inventing sequencing decisions. If yes, skip the plan. If no, write one.

## Plan Structure

Plans live at:

```
docs/superpowers/plans/YYYY-MM-DD-<feature>.md
```

The plan template:

```markdown
# [Feature] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing the end state]
**Architecture:** [2-3 sentences explaining the layering and data flow]
**Tech Stack:** [Key libraries with versions]

**Reference spec:** `docs/superpowers/specs/YYYY-MM-DD-<feature>-design.md`

---

## File Map

| File path                                          | Responsibility                     |
| -------------------------------------------------- | ---------------------------------- |
| `packages/types/src/dtos/tasks.ts`                 | Shared request/response interfaces |
| `apps/api/src/modules/tasks/domain/task.entity.ts` | Domain entity                      |
| ...                                                | ...                                |

---

## Tasks

### Task 1: [Title]

...

### Task 2: [Title]

...
```

The File Map is written before the task list. It forces you to think about ownership — each file appears in exactly one task, which prevents two agents from editing the same file simultaneously.

## Plan Writing Procedure

Follow these six steps in order. Do not skip step 3; dependency analysis is where most sequencing mistakes happen.

1. **Read the spec ACs — each AC needs at least one task.** List every acceptance criterion and mark it as covered. An AC with no corresponding task will not be implemented. An implementation step with no corresponding AC is scope that was never agreed.

2. **Map each AC to files.** For each criterion, identify which files must be created or modified to satisfy it. Use the stack layer list below. Write the File Map from these results. If two ACs touch the same file, the tasks that produce them are sequential.

3. **Identify dependencies.** A task B depends on task A when B imports, calls, or otherwise requires output that A produces. Mark every dependency explicitly. The sequencing rules in this document cover the common cases for this stack — read them before drawing the dependency graph.

4. **Group independent tasks.** Tasks with no shared files and no import relationship can run in parallel. Mark them as a parallel group. They will be dispatched together in the Dispatch phase (see [05-dispatch.md](05-dispatch.md)). Each agent in the group must own exclusive files — if two tasks touch the same file, they are sequential, not parallel.

5. **Estimate complexity: S / M / L — split L tasks.** An S task touches one file and has one verification command. An M task touches two or three files across one layer. An L task touches more than three files, crosses multiple layers, or requires more than one session. Split every L task. A task is too large if a human reviewer cannot evaluate its diff in under ten minutes.

6. **Write each task with file paths, numbered steps, a verification command, and a commit message.** A task that does not specify its verification command requires the executor to invent one — which means it may never be run.

## Task Anatomy

A well-formed task contains everything the executing agent needs: the target files, ordered steps with code, the exact command to verify the result, and the commit message. Nothing is left to interpretation.

````markdown
### Task 3: Define shared task DTO interfaces in packages/types

**Complexity:** S
**Depends on:** none (this is the root of the dependency chain)
**Files:**

- `packages/types/src/dtos/tasks.ts` (create)
- `packages/types/src/dtos/index.ts` (modify — add barrel export)
- `packages/types/src/index.ts` (modify — re-export from dtos)

**Steps:**

1. Create `packages/types/src/dtos/tasks.ts`:

   ```typescript
   export interface ICreateTaskDto {
     title: string;
     description?: string;
   }

   export interface ITaskResponse {
     id: string;
     title: string;
     description: string | null;
     status: "open" | "done";
     userId: string;
     createdAt: string;
   }

   export type ITaskListResponse = ITaskResponse[];
   ```
````

2. Add the barrel export to `packages/types/src/dtos/index.ts`:

   ```typescript
   export * from "./tasks";
   ```

3. Verify the package builds with no type errors:

   ```bash
   cd packages/types && pnpm build
   ```

4. Commit:

   ```
   feat(types): add ICreateTaskDto and ITaskResponse interfaces
   ```

- [ ] Step 1 complete
- [ ] Step 2 complete
- [ ] Verification passed
- [ ] Committed

```

The checkbox list at the bottom is for the executing agent to track progress within the task. A session that is interrupted can resume from the last unchecked step.

## Sequencing Rules for This Stack

The stack has a strict dependency order. Violating it causes import errors that are expensive to untangle mid-implementation.

| Rule | Why |
| ---- | ---- |
| `packages/types` interfaces before NestJS DTO classes | DTO classes `implements` the shared interface — the interface must exist first |
| `packages/types` interfaces before Next.js page types | Page components and API routes type their responses against shared interfaces |
| Drizzle schema (`*.schema.ts`) before use cases | Use cases call the repository; the repository uses `$inferSelect` / `$inferInsert` from the schema |
| Repository interface (domain) before repository implementation (infrastructure) | The implementation `implements` the domain interface |
| Use cases before controllers | Controllers call use cases — a controller cannot be written until the use case signature is known |
| NestJS controller before Next.js BFF route | The BFF route (`apps/web/app/api/<domain>/route.ts`) forwards to NestJS — the endpoint must exist before the proxy is built |
| Next.js BFF route before page component | The page fetches from the BFF route — the route must exist before the page can call it |
| Implementation before tests only when not using TDD | If you are following TDD (red-green-refactor), write the test first. If you are writing tests after implementation, the implementation is the dependency |

## Parallelizable vs. Sequential

Tasks that touch different files in different layers with no import relationship can run simultaneously. Tasks in a dependency chain must run in the order the chain dictates.

| Parallelizable (different file owners, no shared imports) | Sequential (dependency chain) |
| ---------------------------------------------------------- | ----------------------------- |
| `packages/types` DTO interfaces + Drizzle schema definition | `packages/types` interfaces → NestJS DTO class |
| NestJS use case implementation + Next.js page component (UI only, no API calls yet) | Drizzle schema → repository implementation → use case |
| NestJS unit tests + Next.js component unit tests | NestJS controller → Next.js BFF route → page component (with live data) |
| `packages/types` interfaces + `packages/mail` email templates | NestJS controller → Next.js `api.ts` client call → page fetch |
| Multiple independent use cases within the same module | domain entity → repository interface → repository implementation |

When in doubt: grep for `import` paths. If task A produces a file that task B imports, B is sequential to A. If no such import exists, they can run in parallel.

## When to Stop Splitting

Splitting is a tool, not a goal. Over-split plans create coordination overhead — every task boundary is a place where context must be transferred between sessions or agents.

A task is small enough when all three of these hold:

- It touches three or fewer files
- It can be completed and verified in a single agent session without checkpointing
- It has a single verification command that confirms the entire task is done

If a task meets all three criteria, do not split it further. A task that modifies `ICreateTaskDto` in `packages/types`, implements `CreateTaskDto` in the NestJS DTO layer, and runs `pnpm build` to verify both is appropriately scoped — splitting it into two tasks adds a commit boundary with no reduction in risk.

## Anti-patterns

- **Mega-tasks that skip the dependency graph.** A task titled "Implement task management backend" that touches six files across four layers is not a task — it is a mini-spec. The agent has no guidance on sequencing and will make implicit decisions about order that may be wrong. Break it down until each task has a clear first file to create and a clear verification command.

- **Tasks without verification commands.** A task that ends with "commit the changes" without a `pnpm build` or test run step has no pass/fail gate. The agent commits broken code, the next task starts, and the breakage is discovered late. Every task must specify exactly how to verify it is done.

- **Parallelizing tasks that share a file.** Dispatching two agents to simultaneously modify `packages/types/src/dtos/tasks.ts` produces a merge conflict or silent overwrite. Before marking tasks as parallel, check the File Map: each file must appear in exactly one task.

- **Writing the plan before the spec is approved.** A plan derived from a draft spec is a plan for a moving target. When the spec changes (and unapproved specs always change), the plan must be rewritten. Complete the spec review — zero open questions, all ACs binary — before writing a single task.

- **Tasks sized for humans, not agents.** A human developer might spend a day on a task that crosses four files and three layers, context-switching as needed. An agent session has a fixed context window and a finite ability to track partial state. L-sized tasks produce agents that lose track of earlier steps, repeat work, or skip verification because the session is running long. Size tasks for agent sessions, not human workdays.

- **Omitting the File Map.** A plan with tasks but no File Map cannot be reviewed for parallelism conflicts. The File Map is the artifact that lets you prove, at a glance, that two parallel tasks do not collide. Skipping it means you only discover conflicts at dispatch time — or after a destructive overwrite.

→ Next: [05-dispatch.md](05-dispatch.md)
```
