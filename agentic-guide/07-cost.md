# Token Budget Engineering

Token budget engineering is the practice of structuring prompts, routing tasks to appropriately-priced models, and exploiting caching to reduce inference costs without degrading output quality. Done well, it cuts per-feature spend by 20–30% [suggested operational target] while keeping agent behaviour identical to an unoptimised run.

## When to Think About Cost

Cost engineering is not a first-day concern. Build the harness, get agents working correctly, and establish a quality baseline first. Once a feature pipeline is producing reliable output, look at the session token logs and ask whether each token spent was necessary.

Two signals indicate you should audit costs now:

- Your per-feature spend is consistently above $12 [suggested operational target].
- A senior engineer reviews agent sessions and notices repeated full-file reads of files that could have been grepped.

Before optimising, measure. Run three representative features end-to-end, record the session cost for each, and take the average. That number is your baseline. Every decision below should be evaluated against it.

## Model Routing

Anthropic's model family covers a wide price range. Routing each task to the cheapest model that can handle it reliably is the highest-leverage cost lever available.

| Model         | Best for                                                                                                                   | Cost tier                                                                              |
| ------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Claude Haiku  | File enumeration, grep result processing, simple formatting, summarisation, routine data extraction                        | Cheapest [course benchmark: ~$0.25/1k output tokens — verify at anthropic.com/pricing] |
| Claude Sonnet | All implementation tasks: writing code, editing files, authoring specs and plans, reasoning about architecture             | Mid [course benchmark: ~$3/1k output tokens — verify at anthropic.com/pricing]         |
| Claude Opus   | Architectural decisions where Sonnet has failed the same task ≥2 times; novel reasoning problems with no mechanical answer | Highest [course benchmark: ~$15/1k output tokens — verify at anthropic.com/pricing]    |

**Routing rules for this project:**

- **Default to Sonnet** for all implementation tasks. The vast majority of work in this codebase — writing NestJS modules, authoring React components, debugging type errors — is within Sonnet's capability.
- **Escalate to Opus only when Sonnet fails the same task ≥2 times.** If Sonnet's output is rejected twice by quality gates or human review for the same underlying reason, that is a signal the task requires deeper reasoning. One failure is not enough: it may be a bad prompt, not a model-capability ceiling.
- **Use Haiku for preprocessing tasks** that produce structured output consumed by a subsequent Sonnet or Opus call: listing directory contents, extracting line numbers from grep output, reformatting a JSON blob, generating a short summary of a changelog.
- **Never use Opus for tasks with a known, mechanical answer.** Running the Opus model to check whether a file exists, to count lines in a diff, or to reformat an import block is pure waste. The answer is deterministic; Haiku will reach it at a fraction of the cost.

In practice, a well-routed parallel dispatch session looks like this: a Haiku agent enumerates the affected files and extracts relevant line ranges, then three Sonnet subagents each receive a tight context bundle and implement one module in parallel. Opus never fires unless a Sonnet pass fails review twice.

## Prompt Caching

Anthropic's API caches prompt prefixes and reuses them across requests within the TTL window [course benchmark: 5-minute TTL]. When a cache hit occurs, input tokens in the cached prefix are billed at a reduced rate rather than the full input token price. For agent loops that share a large stable context — CLAUDE.md, rules files, skill documents — cache hits accumulate quickly.

**What gets cached vs. what cache-busts:**

| Content                                              | Cache behaviour                                                                           |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| CLAUDE.md contents                                   | Cached on first call; reused for the session duration as long as requests stay within TTL |
| Rules files (`.claude/rules/*.md`)                   | Cached when included in the system prompt prefix                                          |
| Skill documents                                      | Cached when loaded once; reused across subagent calls in the same session                 |
| Current file contents (Read tool output)             | Included in the user turn — busts the cache if the file changes between calls             |
| Tool call results (grep output, bash output)         | Dynamic; always counted as new tokens                                                     |
| Agent-to-agent context passed in a subagent dispatch | New tokens on each dispatch unless the orchestrator sends the same prefix unchanged       |

**How to structure prompts for maximum cache hit rate:**

Put stable content at the top of the prompt, dynamic content at the bottom. The cache prefix must match exactly from position zero to the cache boundary. Any character that changes — even whitespace — at or before a token position invalidates the cache from that point forward.

Concretely: CLAUDE.md and rules files go first. The current task description and dynamic file contents go last. If your orchestrator prepends a timestamp or a run ID to every prompt, move it to the end or remove it; it cache-busts everything that follows.

**Loop timing:** The 5-minute TTL [course benchmark] means a loop iteration that takes longer than 270 seconds will let the cache expire before the next request arrives. Keep inner loop iterations under 270 seconds to stay warm. If a single step genuinely requires more processing time, break it into two sequential requests that each complete within the window.

## Token Budget Audit Procedure

Run this procedure after any session where per-feature cost exceeds your baseline by more than 20%.

1. **Check session cost.** Use the session cost log or API usage dashboard. Record total input tokens, total output tokens, and any cache hit/miss breakdown. Output tokens are priced higher than input tokens — a high output token count often points to verbose agent responses rather than meaningful work.

2. **Identify the top three token consumers.** Sort tool calls or turns by token count. In most sessions, two or three calls account for the majority of tokens. Common offenders: reading a large file to find one function, re-reading the same file multiple times across turns, or passing the full project context to every subagent regardless of relevance.

3. **For each top consumer: could it be a targeted grep instead of a full file read?** If an agent read a 400-line file to find a single function signature, a `grep -n "functionName"` followed by a narrow `Read` with `offset` and `limit` would have used a fraction of the tokens. Targeted reads are almost always sufficient when the agent knows what it is looking for.

4. **For repeated context: is this in CLAUDE.md yet?** If the same background information — a package name, a file path convention, a constraint — appeared in three or more agent turns as part of the prompt, it should be in CLAUDE.md where it gets cached. Paying input token prices for information that never changes is unnecessary.

5. **For multi-agent round trips: did the orchestrator pass unnecessary context to workers?** Each subagent should receive only the context it needs for its specific task. Passing the full orchestrator context to every worker multiplies the per-worker input cost by the number of workers. Audit what each worker actually used, then trim what it ignored.

Target a 20–30% reduction [suggested operational target] in per-feature token spend after a single audit cycle. If you do not reach that threshold, the top consumer in step 2 is likely a structural issue — a pattern baked into how your prompts are assembled — rather than a one-off inefficiency.

## Cost per Feature Benchmark

An unoptimised agent run through the full Perceive → Spec → Plan → Dispatch → Verify workflow costs approximately $17.42 per feature [course benchmark]. This figure assumes Sonnet for all tasks, no caching, full-file reads, and no subagent context trimming.

The optimised target is $8–12 per feature [suggested operational target]. The gap is closed primarily by:

- **High cache hit ratio.** A session where CLAUDE.md and rules files are cached from the first request saves hundreds of input tokens per subsequent request. On a 20-turn session, this alone can account for 30–40% of input token savings.
- **Haiku for preprocessing.** Routing file enumeration and grep processing to Haiku reduces per-task cost by roughly 10x for those steps relative to Sonnet.
- **Targeted file reads.** An agent that reads 50 lines around a grep match instead of the full 400-line file reduces input tokens for that read by 85%.
- **Tight subagent context budgets.** An orchestrator that sends each worker only the files and context specific to its task — rather than a full snapshot of the session — keeps per-worker input costs proportional to the task, not the session.

## Anti-patterns

**Reading whole files to answer targeted questions.** If the question is "what is the return type of this function," reading the entire file is wasteful. Use `grep -n` to find the line, then read a narrow window. The habit of always using a full Read is one of the most common cost anti-patterns in agent sessions.

**Routing non-reasoning tasks to Opus.** Opus is not faster or more accurate on mechanical tasks — it is only more capable on open-ended reasoning problems. Using Opus to reformat a JSON file or check a type signature spends 5–60x more than necessary with no quality benefit.

**Busting the cache with dynamic prefix content.** Adding a timestamp, a random session ID, or a variable task counter at the top of every prompt prevents the stable system prompt content from ever being cached. Move dynamic content to the end of the prompt. This single change can eliminate cache misses entirely for sessions with consistent prefixes.

**Passing full orchestrator context to every subagent.** An orchestrator that has read 15 files to build its plan does not need to forward all 15 files to every worker. Each worker needs the files relevant to its own task. Indiscriminate context forwarding multiplies input costs linearly with the number of subagents.

**Skipping the audit.** Cost optimisation only compounds if you measure. An engineering team that runs sessions without reviewing token logs will not notice when a new prompt template introduces a full-file read on every turn. Schedule a cost audit after every five features and treat regressions as bugs.

**Re-reading files that have not changed.** Agents that re-read the same file at the start of every turn to "refresh context" are paying for information they already have. If file content is stable and already in context, trust it. Only re-read when there is a specific reason to expect the file has changed.

---

→ Next: [08-production.md](08-production.md)
