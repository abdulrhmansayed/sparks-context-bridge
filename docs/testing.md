# Testing the idea

I want technical feedback and real comparisons. This repository has the method, the token receipts and a [small example you can run](reference-example.md). The Spark app and my private chat histories aren't included.

## What you can check today

[The method](method.md) explains A and B, correction links, the map, the incremental index and how requests are built. [The cost note](caching-and-cost.md) explains the pricing. With [the measurement records](../data/README.md) you can recompute every token cost and per-message total yourself.

The receipts give per-inference token counts and rates without prompts, account details or local paths. They let you check my arithmetic. Checking what Spark actually sent, and whether it picked the best sources, needs the tests below.

The example runs offline with hand-written B records. It shows addressing, exact reads, project scope and the rule that an assistant message can't approve anything. The helper model's indexing and source selection are part of Spark itself.

## The tests I'm planning

These are the next comparisons. I'll publish the results whichever app they favor.

### Hidden rules without reminders

Place five project rules early in a realistic long history. For example: no Library X, local-only storage, a 30-second autosave interval, dark mode still only a proposal, and a required keyboard interaction. Later requests should require those rules without naming them or supplying source addresses.

Write the expected behavior before the run. Include a later correction, an unrelated project with a conflicting rule, and tasks where a rule is irrelevant. Score application of the right rule, exact source recovery when requested, current correction handling, and absence of cross-project leakage. A rule appearing in B is not sufficient if the final code violates it.

### Owner-graded real-project questions

The project owner writes ten questions about real work with answers they already know, plus the source or artifact supporting each answer. Freeze that answer key before either arm runs. Remove private details from anything later published.

The owner grades replies against the key, ideally with system names hidden and output order shuffled. Report source accuracy, task accuracy, and uncertain or ambiguous answers separately. Assistant checks can serve as a second audit, but the tested assistant never grades its own answers.

### Five matched coding tasks

Start from identical copies of one repository and frozen requirements. Use five tasks such as implementing a date picker, changing validation, adding keyboard behavior, persisting a setting, and applying a later requirement correction. Match tools, permissions, model setting, and acceptance criteria.

Run each arm against its own copy, with independent checks of the same required behavior. Include setup, helper work, retries, test execution, and any context transition. Record final artifact differences instead of treating two different feature sets as equal success.

### A complete working session

Measure from opening the project until the requested work is verified complete. Keep first text, full reply, tool work, preparation, context transitions, and validation separately observable, and also report the complete elapsed wait.

Include all tasks in the sequence, including failed or interrupted ones. Distinguish user waiting from background work, and avoid adding overlapping operations twice. Alternate arm order across repeated sessions. This tests the practical experience that a short recall quiz cannot establish.

## A useful independent workload

Start both systems from the same files and history. Match the main model, reasoning effort, tools, permissions, and task instructions. Include the setup required by each approach once, then keep the work going.

Use tasks that need old exact text, correction chains, unfamiliar topic combinations, pending decisions, separate project scopes, and actual artifact changes. Include broad questions as well as narrow lookups. An agent that succeeds only when given the exact message ID has not demonstrated that it can find the right evidence on its own.

Let context management happen naturally. If you deliberately trigger compaction or a session reconstruction, label that event and measure it separately. Preserve the originals so someone else can check quotes, decisions, and final work against them.

## Project resumption and context-transition test

The ten-message test's largest cost differences appeared at the cold start and first reply after manual compaction. The next comparison should test how those conditions recur during an actual project.

1. **Prepare one realistic project history and identical starting files.** Use completed work, later corrections, tool results, pending decisions, and short as well as long messages. Match the main model, reasoning setting, tools, permissions, and requested final changes.
2. **Measure a warm baseline.** Continue both arms through several matched tasks before any deliberate break. Record cost, cache reuse, and task completion so ordinary continuation has its own result.
3. **Resume after five one-hour breaks.** Use the same project and comparable new tasks, alternate execution order, and preserve each arm's index and active session. Do not rebuild either arm just to manufacture a cold request. A one-hour pause is a test condition, not an assumption that the cache expires.
4. **Record what actually happened.** Classify each return by observed fresh/cached token usage and any context transition. Distinguish resuming an existing thread from explicitly reopening imported history or deliberately compacting it.
5. **Continue through each system's own context boundary.** Include Spark's preparation and evidence redelivery when it rebuilds, and Codex's actual compaction work. A separate forced-transition test can probe the mechanism, with its conditions labelled clearly.
6. **Score equivalent completed work.** Check historical requirements, correction handling, tests, and final artifacts independently. Include every helper, retry, failed attempt, and blocked wait. Charge initial indexing once, then also show later costs separately.

Report each phase and the complete session total. A result from low cache reuse should not become the expected saving on every warm turn. If no cache loss or natural transition occurs, report that outcome and the measured warm-session result.

## Capturing the compaction price

For each compaction, retain its timing and, when the engine exposes them, its distinct input, cached-read, cache-write, and output counts. Identify the model and any applicable pricing tier, then price that request once. Reconcile it with the workload's complete inference log so it is neither omitted nor added twice.

Do not infer a compaction charge from the next reply's input size. The operation and the reply are different stages. If only cumulative counters are exposed, a delta is usable only when the event boundaries and other inferences are accounted for. If usage remains unavailable, preserve the missing value and report the limitation. The ten-message test's unreported receipt cannot be replaced with the price of a different compaction.

## Comparing practical speed

A recall reply and a whole coding task are timed differently. A smaller context can speed up one step while extra lookup rounds slow down another. In my own side-by-side use, Spark often finishes first, and this test is how I'll measure that properly.

Measure from sending the request until an equivalent verified result is available. Keep time to first text, full reply, tool execution, blocking preparation, context transitions, and final artifact validation separately observable. Count overlapping background work once. Repeat the comparison under warm and low-cache-reuse conditions with alternating run order.

## What to record

| Category | Useful evidence |
|---|---|
| Cost | Per-inference fresh input, cached reads, cache writes, output, model rates, helpers, metered services, and retries. |
| Setup | Initial indexing/preparation cost and time, plus whether earlier setup was reused. |
| Latency | Dispatch to first visible text, full reply, final artifact, and blocking context transitions. |
| Correctness | Exact source fidelity, correction handling, approval boundaries, and independent checks of the artifact. |
| Continuity | What survives a restart or new model session, and which evidence needs redelivery. |
| Scope | Whether unrelated projects influence retrieval or actions. |

Repeat with different task orders and histories. Cache warmth and provider load can affect an individual run. Report distributions as well as averages, and keep required-fact coverage separate from claims of flawless answers.

## What counts as success

The method has to finish the same work for less recurring context while keeping the evidence and meaning needed to continue. A cheaper index is worth nothing if the agent keeps missing the relevant correction, and a cheaper answer is worth nothing if the finished work is wrong.

I also want to understand speed: when less input helps, when extra lookups cost more time than they save, and whether Spark's rebuilds interrupt work less often than Codex's compaction.

If you have a workload or criticism that would test these points, [open an issue](https://github.com/forasm/sparks-context-bridge/issues). Include the comparison conditions and the exact claim you want to examine. Please avoid posting credentials or private conversations.
