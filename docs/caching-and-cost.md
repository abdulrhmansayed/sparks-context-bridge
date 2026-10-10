# Caching, input cost, and continued use

The prices in this repository are **API-equivalent** estimates calculated from recorded tokens at stated model rates. They are not subscription billing records.

## Same model, same cache rates

Spark and Codex are priced at the **same unit rates for the same model**. In the ten-message test, both main agents use the primary-model rates listed below: $2.00 per million fresh input tokens, $0.10 per million cached input tokens, $2.50 per million cache-write tokens, and $10.00 per million output tokens.

Codex's cached input is priced at the cheap cached rate, so caching is already counted in its favor. Spark saves by sending only the history a task needs and by doing its indexing with a much cheaper helper model, whose cost is included.

The cache rate is the price of a cached token. The cache-hit share is how much of the input was actually reused, and that depends on each request. The receipts record fresh and cached tokens separately for every call, so neither side is assumed to have a warm cache.

In the ten-message test, Spark's main model got **89.12%** of its input from cache, against Codex's **81.90%**. The helper model, **Luna**, did all indexing and maintenance for **$0.03240252** combined, and that's in Spark's total.

Counting Luna's calls too, Spark's overall cached share is **67.07%**. That mixes two models with different jobs and prices, so compare the main models with each other. [Main and helper usage separately](measurements.md#cached-input-and-call-counts)

## Four different things that can look like reuse

| Mechanism | What is reused | What still needs to happen |
|---|---|---|
| Stored originals | Exact local evidence | Select and deliver useful sources. |
| Persistent index | Earlier interpretation and navigation work | Process new material and corrections. |
| Continuing model session | Existing conversation state | Perform another inference with its context. |
| Provider prompt cache | Eligible processing of a matching prefix | Process new input and generate the answer. |

The context bridge uses the first two to make project history addressable. It can use continuing sessions and benefit from prompt caching as well. The local database does not create or manage the provider's cache.

## What cached input means

Prompt caching reuses intermediate model states for an unchanged input prefix. Reuse depends on a matching rendered prefix, eligible boundaries, routing, and retention. Changes to earlier instructions, tools, or context can prevent reuse after the changed point. Cached input still carries a charge; output is generated anew. Cache writes have their own rate where applicable, rather than an additional fee on top of another input rate. [OpenAI prompt caching documentation](https://developers.openai.com/api/docs/guides/prompt-caching).

That is why stable instructions and tool definitions belong before changing task material. A changing map or evidence packet should not unnecessarily replace a stable prefix. Actual hits must still be measured from usage counters.

The relevant cache lifetime and controls depend on the model and service configuration. Persistent evidence remains necessary even when a provider cache entry is no longer available.

## Why reduce input if it can already be cached?

Consider a hypothetical rate of $2 per million fresh tokens and $0.10 per million cached tokens. A 150,000-token old prefix costs $0.015 on a fully cached reuse. A 15,000-token old prefix costs $0.0015 under the same assumption. Both are inexpensive compared with fresh processing, but repeating them many times still creates different totals.

Now consider a fresh read. Fifteen thousand useful tokens at that rate cost $0.03, before output. Reopening evidence can therefore cost more than another reuse of a much larger cached prefix. "Fewer tokens" alone does not decide which approach is cheaper. The cache state, model rates, source reads, helper work, and answer length all matter.

These are made-up numbers to show the trade-off the tests measure: whether carrying less history saves more than the extra work of finding and opening evidence costs. In the ten-message test it did, on every message except one.

## Input versus output

A long session can carry large input across many model rounds while producing relatively little visible text. The recurring volume can make input dominate total cost even when output has the higher unit price.

So a statement such as "the input is costing much more than the output in my chat" describes that workload's totals. It does not mean that one input token is priced ten times higher than an output token.

The bridge targets recurring context. It does not make the same answer's output tokens free. A longer answer, extra reasoning, or additional tool decisions can reduce the percentage saving even if historical input is smaller.

## Rates used for the October 10 comparison

Standard text rates per million tokens, checked on **10 October 2026**:

| Model | Fresh input | Cached input | Cache write | Output |
|---|---:|---:|---:|---:|
| Primary model, both apps | $2.00 | $0.10 | $2.50 | $10.00 |
| Luna indexing helper | $0.10 | $0.01 | $0.125 | $0.50 |

Sources: [Primary-model pricing](https://developers.openai.com/api/docs/models/gpt-6.1-sol), [Helper-model pricing](https://developers.openai.com/api/docs/models/gpt-6-luna). Prices and long-context rules can change; these are the rates used for this dated comparison. Exact model identifiers remain in the numeric evidence for reproducibility.

For an inference whose input total includes cached reads and cache writes:

```text
fresh = input_total - cached_read - cache_write

cost = (fresh × fresh_rate
      + cached_read × cached_rate
      + cache_write × write_rate
      + output_total × output_rate) / 1,000,000
```

Apply any applicable context-tier or service multiplier to that inference's rates. Do not average token totals across requests before deciding which price tier applies. No inference in the published October 10 receipt set was marked as long-context pricing.

Reasoning tokens included in an output total must not be charged a second time. The same goes for cumulative session counters: use per-inference deltas or deduplicated receipts, rather than adding a cumulative total after every update.

## Whole-workload accounting

For the bridge:

```text
total_bridge(N) = initial_index
                + sum(main_inferences for N tasks)
                + sum(new indexing and maintenance)
                + sum(other metered retrieval or services)
                + sum(retries and recovery work)
```

For Codex:

```text
total_codex(N) = initial preparation
              + sum(main_inferences for N tasks)
              + sum(actual compactions)
              + sum(other metered services)
              + sum(retries and recovery work)
```

Ordinary local database reads do not create a model-token charge by themselves. Their processing and waiting still matter, and a read followed by another model inference creates model usage.

Charge initial indexing once to the workload that imported the history. A continuing-use table may show new work separately, but should also identify the prior setup. Rebuilding a model session does not justify charging the same historical index again unless a rebuild actually occurred.

Compaction receives the same treatment: count the event when it happens. Do not charge it on every subsequent reply, and do not call an unreported receipt zero.

## Why long projects benefit most

The method is built for long, continuing work. The archive keeps every original, and the model gets the evidence the current task needs. In the ten-message test, the biggest savings came when Codex had to process a large context with little cache reuse: the first message on the imported history (76%) and the first reply after compaction (89%). Normal back-and-forth between those moments saved **20.35%**, and the whole run **60.98%**.

The archive and index survive when the model session changes, so Spark can restore the current work and reopen evidence without replaying every archived message. Spark's own working thread still grows with new messages and tool calls, and the rebuild is designed to keep it in check over a long project.

The one-time import gets cheaper per message the longer you work. A $0.02 import spread over 100 later messages is $0.0002 each. Main-model and maintenance costs still apply to every message.

In a simplified uncached full-history model, a transcript growing by `d` tokens each turn and starting at `H` tokens has repeated historical input proportional to:

```text
N × H + d × N × (N - 1) / 2
```

That is a useful illustration of repeated-history growth, not a forecast of Codex billing. Real sessions use caching, compaction, different tools, varying outputs, and context policies. A bridge also spends input on its map, recent work, and retrieved sources.

Savings can appear on the first message after import. They can also rise or fall as cache conditions and evidence needs change. In the ten-message test, the cumulative saving fell from about 76% at message one to 49% at message nine, then reached 61% after the post-compaction reply. The [phase breakdown](measurements.md#where-the-saving-came-from) and [CSV](../data/long-history-turns.csv) show both the transitions and the warm turns.

For real work, the questions are how often a project has to process a large context fresh, and what Spark's own rebuilds cost. Coming back after a break is the obvious test case. The resumption test records the actual cached-token counts on each return, so it shows whether the cache really expired.

## Cost and latency answer different questions

The time until an answer completes includes local preparation, blocking helper work, main inference, retrieval rounds, tool execution, and any context transition. Background operations that overlap with those stages should not be added as if they all happened sequentially.

A larger cached request can finish before a smaller request requiring multiple round trips. An initial index can be cheap but take noticeable time. A reconstruction or compaction event can create a longer pause even when later replies are quick.

For practical use, I want to measure both the ordinary send-to-answer wait and the interruptions over an entire session. The [measurement note](measurements.md) reports exactly what was timed, including setup and the manual compaction.
