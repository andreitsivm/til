# Quality Gates

Quality gates are automated checks that block work from advancing until a defined standard is met. Unlike a code review comment that can be deferred, a gate is binary: the work passes or it does not proceed.

## When to Set Up Quality Gates

Set up quality gates at the start of a feature, not after it breaks. The earlier a gate exists, the more useful it is as a constraint on the agent — an agent that writes code against an already-passing test suite will write different (better-constrained) code than one working in a vacuum.

Good moments to add gates:

- When starting a new module or service boundary
- When a human reviewer catches the same class of mistake twice
- When integrating a third-party dependency with a complex contract (Paddle webhooks, Auth.js session shape, JWT payload fields)
- Before handing a feature branch to another agent for extension

Gates you already have in this repository (hooks enforced on every file write): Prettier formatting and TypeScript compilation. Gates you should add per feature: passing unit tests, zero ESLint warnings, and a successful production build.

## TDD Workflow with Agents

### The Pattern

1. **Write a failing test.** Express the acceptance criterion as an executable test. The test must fail before any implementation exists — if it passes immediately, the criterion was already met or the test is wrong.
2. **Dispatch an implementation agent.** Give the agent the test file path and the command to run it. The agent's only goal is to make the test pass without modifying the test.
3. **Merge when green.** Do not review implementation style until the test passes. Once it does, run the full suite to confirm no regressions before merging.

### Why This Works With Agents

The test is the acceptance criterion in executable form. Instead of describing desired behaviour in prose — which an agent can partially satisfy, misinterpret, or over-engineer — you give it a program that succeeds only when the behaviour is correct. The agent cannot argue with a failing assertion, cannot ship a partial implementation and call it done, and cannot introduce a subtle semantic shift in interpretation. The test removes ambiguity at the point where ambiguity is most costly: at the boundary between your intent and the agent's execution.

### Example

The example below uses a hypothetical `CreateTaskUseCase` — the `tasks` module does not exist yet, but this shows the pattern you would follow when it is built.

Test file path (to be created when the module is scaffolded):

```
apps/api/src/modules/tasks/application/use-cases/create-task.use-case.spec.ts
```

Run just that test:

```bash
cd apps/api && pnpm test --testPathPattern="create-task"
```

Run the full API test suite:

```bash
cd apps/api && pnpm test
```

A minimal spec follows the Arrange–Act–Assert structure and targets only observable behaviour — what the use case returns and what side effects it produces on the repository interface, not internal state:

```typescript
describe("CreateTaskUseCase", () => {
  it("persists a task and returns the created entity", async () => {
    const repo = new InMemoryTaskRepository();
    const useCase = new CreateTaskUseCase(repo);

    const result = await useCase.execute({
      title: "Buy groceries",
      ownerId: "user-1",
    });

    expect(result.id).toBeDefined();
    expect(result.title).toBe("Buy groceries");
    expect(await repo.findById(result.id)).not.toBeNull();
  });
});
```

Write this test, confirm it fails (`Cannot find module`), then dispatch an agent with the instruction: "Make this test pass. Do not modify the spec file."

## Plan-Verifier Pattern

### How It Works

The plan-verifier pattern adds a lightweight validation step between an agent's output and the next stage of work. The flow is:

```
Human input → Planning agent (produces plan) → Verifier step (CI or hook) → Implementation agent
```

The verifier checks that the plan meets a defined checklist before implementation begins. Typical checks:

- All acceptance criteria are expressed as test assertions (not prose)
- No new public API surface is added without a corresponding interface in `@workspace/types`
- Environment variables introduced in the plan appear in `.env.example` and `turbo.json`
- No use case touches more than one domain boundary

The verifier can be a second agent given the plan as input and the checklist as its prompt, or it can be a CI step that parses the plan document. Either way, the gate is enforced before expensive implementation work starts.

### GitHub Actions Integration

The stub below shows how to wire a verifier step in CI. **This is a stub requiring claude CLI setup** — the `claude` binary must be available in the runner and `ANTHROPIC_API_KEY` must be set as a repository secret.

```yaml
# .github/workflows/plan-verify.yml
# stub requiring claude CLI setup
name: Plan Verification

on:
  pull_request:
    paths:
      - "docs/plans/**"

jobs:
  verify-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Verify plan against checklist
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          PLAN=$(cat docs/plans/${{ github.head_ref }}.md 2>/dev/null || echo "")
          if [ -z "$PLAN" ]; then
            echo "No plan file found — skipping verification"
            exit 0
          fi
          claude --print "Does this plan satisfy the checklist?
          Checklist:
          1. ACs expressed as test assertions
          2. New types defined in @workspace/types
          3. Env vars added to .env.example and turbo.json

          Plan:
          $PLAN

          Reply with PASS or FAIL followed by a one-line reason." \
            | grep -q "^PASS" || (echo "Plan verification failed" && exit 1)
```

## 20-Trace Eval Methodology

[course benchmark — from Husain & Shankar, cited in Neoversity curriculum]

A single agent run is not a reliable signal. Agents are non-deterministic: the same prompt at temperature > 0 will produce different outputs across runs. The 20-trace methodology turns this variance into a measurement by running a prompt 20 times and scoring each result against defined acceptance criteria.

### When to Use It

Use the 20-trace eval when:

- You are calibrating a new agent prompt before wiring it into automation
- A previously reliable agent starts producing inconsistent results and you need to quantify how bad the regression is
- You want to compare two prompt variants and choose the more reliable one
- A stakeholder asks "how often does this work?" and you need a defensible answer

Do not use it for one-off tasks or tasks where the output is deterministic (temperature 0, no tool calls).

### Procedure

1. **Define the prompt.** Write the exact prompt the agent will receive, including all context it normally gets (system prompt, file contents, instructions).
2. **Define acceptance criteria.** Write 3–7 binary checks that a passing trace must satisfy. Each criterion should be verifiable by a script or a second model, not by subjective human judgment.
3. **Run 20 times with temperature > 0.** Use `temperature: 1` or your provider default. Record each output — do not discard runs that look wrong; those are the most informative.
4. **Score each trace.** Apply all acceptance criteria to each trace. A trace passes only if it satisfies every criterion. Record the binary result (pass / fail) and which criteria failed.
5. **Cluster failures.** Group failing traces by which criterion failed. A cluster of 10 failures all failing criterion 3 tells you something specific to fix; scattered single-criterion failures suggest noise.
6. **Calculate pass rate.** `pass_rate = passing_traces / 20`. This is your primary metric.

### Interpreting Results

| Pass rate                            | Interpretation                                        | Action                                                                                      |
| ------------------------------------ | ----------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| ≥ 85% [suggested operational target] | Prompt is reliable for automation                     | Wire into CI or scheduled agent; monitor for drift                                          |
| 60–84%                               | Prompt works but fails too often for unsupervised use | Identify the most common failure cluster; tighten the prompt or add a verifier step         |
| < 60%                                | Prompt is unreliable                                  | Do not automate; redesign the prompt or break the task into smaller, more constrained steps |
| ≈ 50%                                | Random — no better than a coin flip                   | The acceptance criteria may be ambiguous, or the task is under-constrained; revisit both    |

A pass rate measured over 20 traces has meaningful variance — a result of 85% (17/20) is not meaningfully different from 80% (16/20). Treat the threshold as a zone, not a hard line. Re-run with a fresh 20 traces if the result sits within 5 percentage points of a decision boundary.

## Hooks as Quality Gates

Claude Code's hook system runs shell commands at defined lifecycle points. Hooks in this repository run after every file write, enforcing formatting and type correctness before the next agent action proceeds.

### Hooks in This Repository

The following hooks are configured in `.claude/settings.json`:

| Hook        | Trigger                | Command                          | What it enforces                                                |
| ----------- | ---------------------- | -------------------------------- | --------------------------------------------------------------- |
| PostToolUse | Write, Edit, MultiEdit | `npx prettier --write <file>`    | Consistent formatting on every written file                     |
| PostToolUse | Write, Edit, MultiEdit | `node $PWD/.claude/hooks/tsc.js` | TypeScript compilation — no type errors introduced by the write |
| PostToolUse | \* (all tools)         | `jq . > post-log.json`           | Structured log of every tool response for debugging             |

The Prettier hook runs silently (`|| true`) so a file outside Prettier's scope does not abort the session. The `tsc.js` hook runs the TypeScript compiler against the project and will surface type errors immediately after a bad write.

### Adding a Stop Hook

A Stop hook runs when the agent finishes a turn. Use it to enforce that quality checks pass before the session is considered complete. Add the following to `.claude/settings.json` under a `"Stop"` key:

```json
{
  "hooks": {
    "PostToolUse": [...],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "cd apps/web && pnpm lint && pnpm check-types && echo '✓ Quality checks passed'"
          }
        ]
      }
    ]
  }
}
```

This command runs `pnpm lint` (ESLint, zero warnings allowed) and `pnpm check-types` (TypeScript + next-intl typegen) after every agent turn. If either fails, the output is visible in the session log. Note: Stop hooks do not block the session from ending — they are notification hooks, not blocking gates. For hard blocking, use a PreToolUse hook or a CI check.

### Adding a PreToolUse Block Hook

A PreToolUse hook can inspect a tool call before it executes and return a non-zero exit code to block it. Use this to prevent dangerous Bash commands from running in automated sessions.

Add the following to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.command // empty' | grep -qE '(git push --force|railway up|vercel --prod|DROP TABLE|rm -rf /)' && echo 'Blocked: dangerous command' && exit 1 || exit 0"
          }
        ]
      }
    ],
    "PostToolUse": [...]
  }
}
```

This hook reads the Bash command from the tool input, checks it against a list of dangerous patterns, and exits 1 (blocking the tool call) if a match is found. The patterns above cover force pushes, production deploys, destructive SQL, and recursive root deletion. Extend the `grep -qE` pattern with any command you want to prohibit in your automated workflows.

## Anti-patterns

**Testing after the fact.** Writing tests after implementation to hit a coverage number does not give you a quality gate — it gives you documentation of what the code already does. Tests written after implementation cannot catch design mistakes because the design is already committed.

**Treating a passing lint as a quality gate.** Lint catches style and obvious errors. It does not verify that the feature works, that the API contract is honoured, or that a downstream consumer is unaffected. Use lint as one layer in a stack of gates, not as the gate.

**Running evals at temperature 0.** A deterministic run tells you the output for one seed. It does not tell you how the prompt behaves across the distribution of inputs and model states it will encounter in production. Always run evals at temperature > 0 if the production prompt uses temperature > 0.

**Skipping gates under time pressure.** Gates exist precisely because time pressure increases the rate of mistakes. "We'll add tests after the launch" is a commitment that is rarely kept. The technical debt compounds with every subsequent agent run that builds on the untested foundation.

**Over-specifying hooks.** A hook that runs a 3-minute build on every file write will be disabled or bypassed. Hooks must be fast enough to run without friction — under 5 seconds for PostToolUse hooks. Reserve slow checks (full build, integration tests) for CI on push.

**Conflating pass rate with correctness.** An 85% pass rate means 17 out of 20 randomly sampled traces met your criteria. It does not mean the agent is correct 85% of the time on your production inputs. The eval is only as good as the acceptance criteria and the prompt diversity in your 20 runs.

---

→ Next: [07-cost.md](07-cost.md)
