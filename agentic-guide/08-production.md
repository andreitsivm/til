# Scale & Production Rollout

Scale and production rollout is the process of moving the agentic harness from a single developer's local configuration to a shared, version-controlled system that a whole team operates with consistent guardrails, CI integration, and measurable outcomes. A harness that is not packaged, documented, and monitored degrades as the team grows — plugins drift, costs creep, and no one can tell whether agents are actually accelerating delivery.

## When to Read This Doc

Read this document when you are ready to share the harness with a second engineer, when you want to add CI agents that run autonomously on pull requests or push events, or when you need to establish team-wide metrics to prove — or disprove — that the agentic workflow is worth the overhead. If you are still setting up the harness for the first time, start with [01-harness.md](01-harness.md).

---

## Plugin Packaging

### What a Plugin Contains

A plugin is a self-contained folder that a skills manager (such as the Claude Code marketplace) can install into `.claude/skills/`. Everything needed to invoke the skill must live inside this folder — no external files, no path assumptions relative to the project root.

```
<plugin-name>/
  SKILL.md           ← frontmatter + implementation content
  assets/            ← optional: diagrams, reference tables, example files
  README.md          ← optional: human-readable description for the marketplace listing
```

The only required file is `SKILL.md`. The `assets/` directory is appropriate when the skill references a diagram or a boilerplate file that agents need to read during execution. Do not add build scripts, package manifests, or compiled artefacts — skills are plain Markdown, not packages.

### How This Repository Uses Plugins

All installed plugins are recorded in `skills-lock.json` at the repository root. This file is the single source of truth for which external plugins are active, where they were sourced from, and the hash of the content at install time. Committing it ensures every team member and every CI runner gets an identical set of skills.

The file structure looks like this:

```json
{
  "version": 1,
  "skills": {
    "paddle-billing-history": {
      "source": "PaddleHQ/paddle-agent-skills",
      "sourceType": "github",
      "skillPath": "skills/billing-history/SKILL.md",
      "computedHash": "da635f4393053ca1b23b03214877c8b2ce63bd3e4418a7aa547a90d66708debc"
    },
    "deploy-to-vercel": {
      "source": "vercel-labs/agent-skills",
      "sourceType": "github",
      "skillPath": "skills/deploy-to-vercel/SKILL.md",
      "computedHash": "03e0eaaa9bf13ba1e7ffa387f5893de6f324c0868c627001f179395a8feaa7c9"
    },
    "nestjs-best-practices": {
      "source": "Kadajett/agent-nestjs-skills",
      "sourceType": "github",
      "skillPath": "SKILL.md",
      "computedHash": "1b6f82e889d19d305e38e35594de08eca0242321f353cafa4cf5e61dd3aa1a73"
    }
  }
}
```

Each entry records the GitHub organisation and repository (`source`), the path within that repository to the skill file (`skillPath`), and the SHA-256 hash of the content at the time of installation (`computedHash`). The hash lets you detect upstream drift — if the source repository updates the skill, the hash will not match and you can decide whether to accept the update.

This repository currently has 19 plugins installed across three vendor groups:

- **Paddle group (10):** `paddle-billing-history`, `paddle-catalog-setup`, `paddle-checkout-web`, `paddle-customer-portal`, `paddle-pricing-pages`, `paddle-sandbox-testing`, `paddle-subscription-cancel`, `paddle-subscription-sync`, `paddle-subscription-update`, `paddle-webhooks`
- **Vercel group (6):** `deploy-to-vercel`, `vercel-composition-patterns`, `vercel-optimize`, `vercel-react-best-practices`, `vercel-react-view-transitions`, `web-design-guidelines`
- **General / Misc (3):** `nestjs-best-practices`, `shadcn`, `writing-guidelines`

### Distributing Your Own Plugin

If you have written skills that would be useful to other teams, you can publish them as a plugin on GitHub so others can install them the same way this repository installs the Paddle and Vercel skill sets.

1. **Create a public GitHub repository.** The conventional name is `<org>/<topic>-agent-skills`. Place each skill in `skills/<skill-name>/SKILL.md`. If you have a single skill, `SKILL.md` at the repository root is also accepted by the installer.
2. **Validate all frontmatter blocks.** Every `SKILL.md` must have `name`, `description`, and at minimum one trigger phrase in the `description` field. Missing frontmatter causes the installer to skip the file silently.
3. **Write a `README.md` at the repository root** that lists each skill, its trigger conditions, and any environment variables or MCP servers it depends on. This is what appears in the marketplace listing.
4. **Tag a release.** The skills installer pins to a tag, not a branch, so consumers get a reproducible installation. Use semantic versioning: `v1.0.0` for the initial release, `v1.1.0` for additions that do not break existing triggers, `v2.0.0` for any change to trigger phrases or required environment variables.
5. **Announce the `source` and `skillPath` values.** Tell consumers exactly which path to reference in their `skills-lock.json` entry. Ambiguity here causes installation failures that are hard to diagnose.

---

## CI Agents

CI agents are agent invocations triggered by GitHub Actions events rather than by a human in a terminal. They run the same harness — the same skills, the same MCP registrations, the same hooks — but in an ephemeral runner without a human in the loop. Because they operate autonomously and post results to the repository, the constraints on what they may do are tighter than for interactive sessions.

### Constraints on CI Agents

| Must                                                      | Must not                                                                               |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Run in read-only or append-only mode by default           | Write to `main` directly                                                               |
| Exit with a non-zero code when a quality gate fails       | Suppress errors with `continue-on-error: true` or swallow non-zero exits               |
| Post findings as pull request comments or job summaries   | Send output only to logs where it is invisible to reviewers                            |
| Authenticate via repository secrets (`${{ secrets.* }}`)  | Inline API keys, tokens, or `AUTH_SECRET` values in workflow files                     |
| Record token cost in the job summary for every run        | Run without cost tracking                                                              |
| Run on ephemeral runners that are destroyed after the job | Persist session state, credentials, or agent context between jobs                      |
| Scope MCP server access to read-only tools                | Call destructive Paddle, Vercel, or Railway operations without explicit human approval |

### Use Cases in This Repository

| CI Agent      | Trigger                                    | What It Checks                                                                                                                                                                                                                    |
| ------------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Plan verifier | `pull_request` — on any PR targeting `dev` | Reads the spec and plan files changed in the PR, invokes a `work-planner` subagent to verify that every acceptance criterion in the spec has a corresponding step in the plan, and posts a summary comment if gaps are found      |
| Cost audit    | `push` to `main`                           | Reads the session cost log committed with the merge, calculates per-feature token spend, flags any session that exceeded $12 [suggested operational target], and posts the result to the job summary                              |
| Eval pipeline | Scheduled — weekly on Sunday at 02:00 UTC  | Replays the five canonical benchmark tasks through the full Perceive → Spec → Plan → Dispatch → Verify workflow, records token cost and output quality scores, and commits the results to `docs/eval-results/` for trend tracking |

### Minimal CI Agent Example

The plan verifier slots in alongside the existing `pr-agent.yml` workflow. Save it as `.github/workflows/plan-verifier.yml`:

```yaml
name: Plan Verifier

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - "docs/**"
      - ".claude/skills/**"

jobs:
  verify_plan:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Run plan verifier agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          AUTH_SECRET: ${{ secrets.AUTH_SECRET }}
        run: |
          claude --agent work-planner \
            --prompt "Read the spec and plan files modified in this PR. \
              Verify that every acceptance criterion in the spec has a \
              corresponding step in the plan. List any gaps. \
              Output only a markdown summary — do not write any files." \
            --output-format markdown >> $GITHUB_STEP_SUMMARY

      - name: Post comment if gaps found
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '**Plan verifier found spec–plan gaps.** See the job summary for details.'
            })
```

Key points: the agent runs with `--output-format markdown` so its output is human-readable in the job summary. The workflow fails only when the agent exits non-zero, which it does only when it finds uncovered acceptance criteria. Cost tracking is implicit — the Anthropic API logs token usage against the API key automatically; pipe job summary output to your cost dashboard to complete the audit trail.

---

## 30-Day Rollout Plan

The goal of the rollout is to get a team of engineers using the harness reliably without a drop in delivery speed. The plan is deliberately paced: one week to prove the harness works for one person, one week to prove it works across two, one week to roll out to everyone, and one week to measure and adjust.

### Week 1 — Solo Pilot

**Goal:** Demonstrate that one engineer can run a full feature through the five-phase workflow faster than without the harness.

- Install all 19 plugins from `skills-lock.json` on the pilot engineer's machine.
- Run one real feature — not a toy — from Perceive through Verify using the full workflow. Record the wall-clock time and per-feature token cost.
- Identify any skill gaps: decision points where the agent deliberated because no skill covered the situation. Write a new skill for each gap found.
- Verify that all hooks fire correctly by checking that Prettier and the TypeScript checker run on every file save during the session.
- Confirm that `skills-lock.json` is committed and that a second checkout of the repository installs an identical set of skills.

**Deliverable:** One merged feature PR, one session cost baseline, and a list of any skill gaps that were addressed.

### Week 2 — Pair Pilot

**Goal:** Prove that two engineers can use the harness independently without configuration drift between their machines.

- Onboard a second engineer using only the committed configuration: `skills-lock.json`, `.mcp.json`, `.claude/settings.json`, and `CLAUDE.md`. No verbal setup instructions should be required.
- Each engineer runs one feature independently. Compare the session costs and note any divergence.
- Review both PRs together for agent-output quality: are type errors absent? Do implementations follow the patterns in `.claude/rules/`? Are response DTOs implementing shared interfaces from `@workspace/types`?
- Pair on any configuration inconsistency and update the committed files so both machines converge.
- Enable the plan verifier CI agent on the repository so it runs on all new PRs from this point forward.

**Deliverable:** Both engineers onboarded, plan verifier active in CI, configuration drift resolved.

### Week 3 — Team Rollout

**Goal:** Bring all engineers onto the harness with a shared onboarding checklist.

- Write a one-page onboarding checklist that covers: cloning the repository, installing the skills manager, verifying that all 19 plugins install correctly, running a smoke-test feature through Perceive → Verify, and confirming that hooks fire on file save.
- Onboard remaining engineers in pairs, not solo. Pairing catches configuration issues that a solo engineer might work around rather than fix.
- Hold a 30-minute calibration session where the team reviews one agent session together — walking through the five phases, the skill invocations, and the token cost breakdown — to build a shared mental model of what a well-run session looks like.
- Enable the cost audit CI agent on the repository.
- Agree on the team's operational targets for the four core metrics: per-feature token cost, CI pass rate on first push, plan adherence rate, and rework rate. Record these targets in a short team document.

**Deliverable:** All engineers onboarded, both CI agents active, team operational targets documented.

### Week 4 — Metrics Review

**Goal:** Measure whether the harness is delivering on its promises and identify the highest-leverage improvement.

- Pull data for all four core metrics from the four-week period. Use the cost audit job summaries for token cost, GitHub's CI data for pass rates, PR review comments for rework rate, and time-to-PR for spec-to-plan time.
- Present the results to the team in a structured retrospective. For each metric, compare the current value against the target agreed in Week 3. Label any value that has not improved a regression — not a neutral outcome.
- Identify the single highest-leverage change. Common findings: a missing skill causing repeated deliberation, a hook that is not firing because its matcher is too narrow, or a subagent dispatch pattern that is bloating context instead of isolating it.
- Ship the highest-leverage change before the retrospective ends so the team leaves with a concrete improvement, not only a diagnosis.

**Deliverable:** Four-week metrics report, highest-leverage improvement shipped and committed.

---

## 10 Measurable Metrics

These metrics are the authoritative indicators of harness health. Measure all ten from Week 4 onward. Metrics marked [suggested operational target] are directional targets based on observed performance in similar projects — your baseline may differ.

| #   | Metric                                      | How to Measure                                                                                       | Target Direction                                                                                  |
| --- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1   | **PR cycle time**                           | Time from branch creation to merge, averaged across all feature PRs in the period                    | Decrease; [suggested operational target]: ≤2 days for a standard feature                          |
| 2   | **CI pass rate on first push**              | Percentage of PRs where all CI checks pass on the first push, with no follow-up fix commits          | Increase; [suggested operational target]: ≥80%                                                    |
| 3   | **Rework rate**                             | Percentage of PR review comments that require the author to re-implement logic (not just formatting) | Decrease; [suggested operational target]: ≤15% of review comments trigger rework                  |
| 4   | **Cache hit ratio**                         | Proportion of input tokens served from cache, reported in the Anthropic API usage dashboard          | Increase; [suggested operational target]: ≥60% cache hits per session                             |
| 5   | **Token cost per feature**                  | Total session token cost for a single feature from Perceive to merged PR                             | Decrease; [suggested operational target]: $8–12 per feature after optimisation                    |
| 6   | **Spec-to-plan time**                       | Wall-clock minutes from the start of spec writing to a committed, reviewed plan                      | Decrease; [suggested operational target]: ≤45 minutes for a standard feature                      |
| 7   | **Plan adherence rate**                     | Percentage of plan steps that were executed as written without mid-task revision by the agent        | Increase; [suggested operational target]: ≥85% of steps executed as planned                       |
| 8   | **Test coverage delta per PR**              | Change in line coverage from the base branch to the PR head, measured by the test runner report      | Increase or hold; coverage must not regress below base                                            |
| 9   | **Agent-assisted task ratio**               | Percentage of merged PRs in the period where at least one agent session was used for implementation  | Increase; target adoption across the whole team, not only early adopters                          |
| 10  | **Onboarding time to first AI-assisted PR** | Calendar days from a new engineer's first day to their first merged PR that used an agent session    | Decrease; [suggested operational target]: ≤3 days for an engineer already familiar with the stack |

Collect metrics 1, 2, 3, 8, and 9 from GitHub's pull request and check-run APIs. Collect metrics 4 and 5 from the Anthropic API usage dashboard or the cost audit job summary. Collect metrics 6, 7, and 10 manually — they require session logs or direct observation. A spreadsheet updated after every sprint is sufficient for a team of ten or fewer; automate collection only after you have established stable definitions for each metric.

---

## Anti-patterns

**Treating `skills-lock.json` as optional.** Without a committed lock file, each engineer's local skills installation diverges. One engineer has an updated `paddle-checkout-web` skill; another has last month's version. Agent behaviour differs between machines and CI produces results that cannot be reproduced locally. Commit the lock file and treat updates to it the same as dependency version bumps.

**Enabling CI agents before the harness is stable locally.** If agents are producing inconsistent output on developer machines — misusing patterns, ignoring hooks, generating type errors — enabling CI agents will surface failures at the wrong layer. Fix local harness quality first. A CI agent that catches real quality issues is valuable; one that flags noise from an under-configured harness teaches engineers to ignore its output.

**Setting operational targets before measuring a baseline.** Week 3 asks the team to agree on targets. It is tempting to set aspirational numbers without data. Targets set before a baseline is established are guesses. Run at least two weeks of instrumented sessions before committing to a number. A target that turns out to be trivially easy or structurally impossible both fail to motivate the right behaviour.

**Onboarding engineers solo.** Solo onboarding allows engineers to work around configuration problems silently — they find a way to make the session work without understanding why the standard configuration failed. Pair onboarding surfaces configuration issues in real time and produces fixes that improve the shared configuration for everyone.

**Treating plugins as set-and-forget.** Upstream plugin repositories update their skills. If a vendor changes their API or their recommended patterns, the installed plugin may no longer match current best practices. Schedule a plugin review every four to eight weeks: compare `computedHash` values in `skills-lock.json` against the current upstream content and deliberately accept or reject each change.

**Conflating token cost reduction with quality reduction.** The cost metrics in this guide aim for $8–12 per feature [suggested operational target], but that target is only valid when output quality remains constant. If the team hits the cost target by disabling verification steps, switching to a less capable model for reasoning tasks, or skipping spec reviews, the cost savings are illusory — they show up later as rework, bugs, or support incidents. Track metrics 1–3 and 7–8 alongside metric 5 to ensure cost improvements are not coming at the expense of quality.

---

For the complete reference — harness setup, perception strategies, spec writing, plan structure, dispatch patterns, verification gates, and cost engineering — return to the [index](00-index.md).
