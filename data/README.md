# Published numeric evidence

This folder contains sanitized measurement records. It contains no app code, real conversation text, prompts, model replies, credentials, account IDs, local filesystem paths, or native thread IDs.

| File | Contents |
|---|---|
| [long-history-10-turns.json](long-history-10-turns.json) | The ten-message 63-versus-61 test: workload boundaries, timing and score summaries, model rates, and 45 per-inference token receipts. |
| [long-history-turns.csv](long-history-turns.csv) | Ten turn-level costs and reply times, plus cumulative totals including initial Spark indexing once. |
| [spark-update-10-turns.json](spark-update-10-turns.json) | Latest paired Spark update, its separate earlier Codex reference, aggregate stage counters, scores, timing, and recomputed arithmetic. |
| [historical-observations.json](historical-observations.json) | Selected earlier aggregate measurements and their setup/comparison boundaries. |

## Reading the ten-message test

`inputTokens` includes cached-read and cache-write tokens. Subtract those categories to calculate fresh input, then price all four categories once using the receipt's model. [Cost formula](../docs/caching-and-cost.md).

Each receipt belongs to the initial index, a main reply, or maintenance. Maintenance is associated with the new user turn it processed. The per-turn CSV's Spark main and maintenance columns add up to new work for that turn; the cumulative total begins with $0.02061608 of initial indexing.

There are 16 main Spark inferences, 9 initial helper inferences, 10 maintenance helper inferences, and 10 Codex inferences. Summing the receipts produces **$0.30937612** for Spark and **$0.79289520** for the recorded Codex replies.

Codex's manual-compaction price is represented by `null` and `usageReported: false`. It is not a zero-cost receipt. Its wait is separate from reply latency.

The history hash identifies the frozen synthetic corpus used by the local test. It does not expose that corpus or make the private application execution independently reproducible. The public counters support arithmetic review; answer quality and payload selection require additional evidence or an independent test.

## Provenance and interpretation

The ten-message test was reduced from the completed October 10 long-context test and its separately recomputed audit. Only the successful paired workload's recorded receipts are used in its comparison. Development experiments are outside that workload comparison.

The latest update export is a separate workload. It compares the previous and updated Spark builds, both with fresh indexes and helper work included. Its Codex results are an earlier reference with a different corpus salt, not a concurrently rerun arm. Every arm in that export passed 63/63 required checks. Aggregate stage costs, means, lifecycle totals, and percentage changes were recomputed from the source counters before export; it does not publish a new per-inference receipt set or establish independent replication. Source hashes identify the local comparison and earlier reference without exposing local paths or conversations.

The historical records retain their own measurement units and baselines. Costs are not repriced to merge them into one overall average, and a previously built index is identified explicitly. Some records are assistant-reviewed observations rather than complete public inference traces.

See [the measurement note](../docs/measurements.md) for task scope, setup, timing, quality, and what remains unknown.
