# Spec-Driven Development

A spec is a written contract that defines what an agent must produce and how success will be measured — before a single line of code is written. It separates the question of _what to build_ from _how to build it_, so that agents receive bounded, testable instructions rather than open-ended prompts.

## When to Write a Spec

Write a spec whenever a task involves:

- More than one file being created or modified
- Any decision about what user-visible behavior should look like
- Work that will be split across multiple agents or sessions
- Features that touch the shared `packages/types` contract between NestJS and Next.js

Skip the spec for pure mechanical tasks with zero ambiguity: renaming a variable, bumping a version number, reformatting a file, or applying a pre-defined scaffold exactly as documented. If you find yourself adding a bullet to "skip" reasoning for a task, the task probably needs a spec.

## Why Specs First

Without a spec, an agent fills every undefined edge with a plausible assumption from training data: it invents the field names, decides the endpoint shape, chooses where to put the component, and inlines business rules that were never discussed. Each assumption is locally reasonable and globally wrong. By the time the output surfaces, the scope has drifted, duplicated logic exists in two layers, and the acceptance criteria live only in the reviewer's head — making every review a renegotiation of requirements rather than a check against a written standard.

## The Spec Template

Every spec in this repository follows the same five-section format. Specs live at:

```
docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md
```

The template:

```markdown
# Design: <Feature Name>

**Date:** YYYY-MM-DD
**Status:** Draft | Approved for implementation

---

## Problem

[One or two sentences. What is broken, missing, or needed? Why does it matter now?]

---

## What We're Building

[A tight description of the deliverable — not a list of implementation steps. What will exist after this work that does not exist today?]

---

## Acceptance Criteria

1. [Binary, user-observable, testable. See rules below.]
2. ...

---

## Out of Scope

- [Explicit exclusion — things the reader might reasonably assume are included but are not.]
- ...

---

## Open Questions

- [Anything that must be resolved before implementation begins. No unresolved questions at approval time.]
```

The "Out of Scope" section earns its weight: it is the fastest way to stop scope creep before it starts. If a reviewer says "shouldn't we also…", the answer is "it's in Out of Scope" — not a design discussion.

## Acceptance Criteria Rules

Each acceptance criterion is a test case in prose form. Apply these four rules to every criterion before marking a spec approved:

| Rule                   | Good example                                                         | Bad example                                                       |
| ---------------------- | -------------------------------------------------------------------- | ----------------------------------------------------------------- |
| **Binary (pass/fail)** | `POST /tasks returns 201 with the created task id in the body`       | `POST /tasks should work correctly`                               |
| **User-observable**    | `The task list page at /tasks renders all tasks returned by the API` | `The TaskRepository correctly calls Drizzle with the right query` |
| **Independent**        | `Deleting a task removes it from the GET /tasks response`            | `After steps 1–3 above, the task count decreases`                 |
| **Specific**           | `An unauthenticated request to POST /tasks returns 401`              | `Authentication is handled properly`                              |

A criterion is binary when a person who did not write the spec can evaluate it — unambiguously, with no judgment calls — and return PASS or FAIL. If the evaluator needs to decide what "should work" means, the criterion is not binary.

A criterion is user-observable when it describes a state or behavior visible through the API surface, the UI, or an external integration — not an internal implementation detail like a private method call or a database row shape.

## What Makes a Spec Implementable

Before handing a spec to an agent, verify all of the following:

- **Ten ACs or fewer.** More than ten usually signals that multiple features have been conflated into one spec. Split them.
- **Zero ambiguous nouns.** Every entity name in the spec must be defined. "A task" must have its fields listed or reference a type in `packages/types`. "The user" must be defined as either the authenticated session user or a specific role.
- **All external dependencies named.** If the feature requires a Paddle webhook, name the event. If it requires a NestJS module that does not exist yet, name the prerequisite task. Agents cannot discover external dependencies from context — they must be stated.
- **Out of Scope section present and non-empty.** A spec with no exclusions is an invitation for scope expansion. If nothing is out of scope, the feature is probably under-specified.
- **No open questions at approval time.** Open Questions are a staging area for uncertainty during drafting. A spec is not ready to implement until every open question has been resolved — moved to a decision in another section or explicitly closed as "not needed."

## Spec Self-Review Checklist

Run these four checks on every spec before marking it Approved:

1. **Placeholder scan.** Search the document for the words "TBD", "TODO", "later", "coming soon", and "…". Any match means the spec is not complete.

2. **Consistency check.** Read the ACs against "What We're Building." Every criterion must be traceable to a deliverable named in that section. ACs that test behavior not described in the deliverable are orphaned — they either reveal missing scope or belong in a different spec.

3. **Scope check.** Read the ACs against "Out of Scope." If an AC tests behavior that is listed as out of scope, the lists are contradictory. Resolve before approving.

4. **Ambiguity check.** For each noun in each AC, ask: does a new reader know exactly what this refers to? If not, define it in the AC or in a preceding section. Pay particular attention to pronouns and generic words like "it", "the record", "the response", and "valid".

## Worked Example

### The Bad Spec

```markdown
# Design: Task Management

## Problem

We need task management.

## What We're Building

A task system where users can create and view tasks.

## Acceptance Criteria

1. Users can create tasks.
2. The task list should display tasks.
3. API should handle tasks properly.
4. Make sure it's secure.

## Out of Scope

N/A

## Open Questions

- What fields does a task have?
- Do we need pagination?
```

This spec fails every rule. AC 1 ("Users can create tasks") is not binary — pass or fail depends on whoever reads it. AC 3 ("handle tasks properly") is not specific and not user-observable. The out-of-scope section is absent. Two open questions remain unresolved. An agent given this spec will invent the field list, pick an endpoint shape, decide whether to add auth, and produce output that is difficult to evaluate because there is no written standard to evaluate it against.

### The Good Spec

```markdown
# Design: Task Management — Create and List

**Date:** 2026-06-09
**Status:** Approved for implementation

---

## Problem

There is no way to create or retrieve tasks in the system. Engineers building the task list page at /tasks have no backend endpoint to call.

---

## What We're Building

A NestJS endpoint to create tasks and a NestJS endpoint to list tasks, plus a Next.js page at /tasks that displays the list. The shared DTO interface for Task lives in `packages/types/src/dtos/tasks.ts`.

A Task has: `id: string`, `title: string`, `description: string | null`, `status: "open" | "done"`, `userId: string`, `createdAt: Date`.

---

## Acceptance Criteria

1. `POST /tasks` with a valid request body (`title: string`, optional `description: string`) and a valid JWT returns 201 with `{ id, title, description, status, userId, createdAt }`.
2. `POST /tasks` without a JWT returns 401. No task is created.
3. `POST /tasks` with a missing `title` field returns 400.
4. `GET /tasks` with a valid JWT returns 200 with an array of tasks belonging to the authenticated user, ordered by `createdAt` descending.
5. `GET /tasks` without a JWT returns 401.
6. The page at /tasks fetches from `GET /tasks` via the BFF pattern and renders the task list. When there are no tasks, it renders the text "No tasks yet."
7. The page at /tasks is accessible only to authenticated users — unauthenticated visitors are redirected to /login.

---

## Out of Scope

- Task update (PATCH /tasks/:id)
- Task deletion (DELETE /tasks/:id)
- Pagination of the task list
- Filtering or sorting tasks by any field other than createdAt
- Task assignment to users other than the creator

---

## Open Questions

(none — all resolved at spec review on 2026-06-09)
```

Every AC in the good spec is binary: a test can be written directly from AC 1 — send the request, assert 201, assert the shape. AC 6 specifies the empty-state text, removing the judgment call. The out-of-scope section answers the first five questions a reviewer would raise. The field list is written into the spec so the agent never guesses.

## Anti-patterns

- **Specs that describe implementation.** "Use a Redis cache to store the task list" belongs in the plan, not the spec. A spec defines what the user observes, not how the system achieves it. Mixing the two couples the acceptance criteria to a specific implementation and makes it impossible to verify behavior without reading the code.

- **ACs that test internal state.** "The `TaskRepository.create` method is called once per request" tests a private implementation detail, not user-observable behavior. If the implementation changes to batch-insert, this AC fails even when behavior is correct. Write ACs against the API surface, the UI, or the external integration — not against function call counts or database queries.

- **Specs written after implementation.** A spec written to match code that already exists is a description, not a contract. It cannot catch scope drift because the scope has already been decided. It cannot surface open questions because questions were answered in code. Post-hoc specs provide false process coverage without the actual benefit — write the spec first so it does the work it exists to do.

- **Skipping the spec for "quick" tasks.** A task feels quick at the moment of dispatch. The cost of a missing spec does not appear until review, when an agent has added a feature that was out of scope, omitted a check that seemed obvious, or chosen an endpoint shape that conflicts with an existing pattern. The spec is not overhead — it is the instrument that makes output reviewable in minutes rather than hours.

→ Next: [04-plan.md](04-plan.md)
