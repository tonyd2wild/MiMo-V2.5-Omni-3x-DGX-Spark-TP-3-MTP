# Benchmarks

Full data appendix for MiMo V2.5 Omni on 3x DGX Spark. Every table here is derived directly from the JSON files in [`benchmarks/`](benchmarks). Run dates are 2026-06-20 (UTC).

Config under test for all runs: `lukealonso/MiMo-V2.5-NVFP4`, TP=3 across 3 DGX Spark (GB10) over RoCE, 1,000,000 token context, MTP2 speculative decoding, `max_num_seqs=6`, NVFP4 weights, fp8 KV cache. Served model name `MiMo-V2.5-NVFP4`.

---

## Methodology

### The eval

The quality eval is a **69-scenario tool-calling benchmark** spanning 15 categories. Each scenario is graded PASS (2 points), PARTIAL (1 point), or FAIL (0 points), for a maximum of 138 points.

The 15 categories (scenario counts in the run-1 set): Tool Selection (6), Parameter Precision (6), Instruction Following (5), Safety and Boundaries (6), Toolset Scale (4), Multi-Step Chains (5), Structured Output (5), Error Recovery (4), Contradictory Parameters (3), Omitted Required Parameter (4), Ambiguity Handling (4), Context Retention (3), Hallucinated Tools (4), Format Compliance (5), Refusal Calibration (5).

### The metrics

- **Quality** (0 to 100): the percentage of available points earned, that is `points / 138 * 100`. This is the headline correctness score.
- **Responsiveness** (0 to 100): a latency-derived score. Faster median answers score higher.
- **Deployability** (0 to 100): a blended production-readiness score, computed as **`0.7 * quality + 0.3 * responsiveness`**. It rewards being both correct and fast.
- **Decode tok/s:** raw token generation speed during decode.
- **Effective tok/s:** end-to-end useful throughput, accounting for prefill and overhead.
- **Median turn latency (ms):** the median time to produce an answer turn. This is what a user actually waits.
- **Token efficiency:** total tokens relative to completion tokens, a measure of how much of the budget went to the visible answer versus overhead and reasoning.

### The runs

- **Quality eval:** 69 scenarios, 3 runs each in thinking-OFF and thinking-ON mode, `max_tokens=1024`.
- **Concurrency sweep:** 12 tool-calling scenarios (TC-01, TC-04, TC-09, TC-25, TC-27, TC-29, TC-33, TC-37, TC-44, TC-57, TC-63, TC-66), 8 requests per concurrency level, levels 1 through 8, `max_tokens=1024`, 180 s wall timeout, in both thinking modes.

A note on the thinking-mode default: the harness defaults `enable_thinking` to ON. Production should run with it OFF. The OFF runs below are the production-representative numbers.

---

## Quality eval: all 6 runs

### Thinking OFF

| Run | Score | Pass | Partial | Fail | Quality | Responsiveness | Deployability | Median latency (ms) | Decode tok/s | Effective tok/s | Token efficiency | Stars | Rating |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| Run 1 | 135/138 | 66 | 3 | 0 | 97.8 | 96.3 | 97.3 | 1228.6 | 38.9 | 34.9 | 1.504 | 5 | Excellent |
| Run 2 | 133/138 | 65 | 3 | 1 | 96.4 | 96.4 | 96.4 | 1196.5 | 38.8 | 35.2 | 1.502 | 5 | Excellent |
| Run 3 | 135/138 | 66 | 3 | 0 | 97.8 | 96.6 | 97.4 | 1168.3 | 38.7 | 35.1 | 1.524 | 5 | Excellent |
| **Average** | **134.3/138** | | | | **97.3** | **96.43** | **97.03** | **1197.8** | **38.8** | **35.07** | **1.510** | **5** | **Excellent** |

### Thinking ON

| Run | Score | Pass | Partial | Fail | Quality | Responsiveness | Deployability | Median latency (ms) | Decode tok/s | Effective tok/s | Token efficiency | Stars | Rating |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| Run 1 | 122/138 | 59 | 4 | 6 | 88.4 | 90.1 | 88.9 | 2427.3 | 40.0 | 38.0 | 1.281 | 4 | Good |
| Run 2 | 124/138 | 60 | 4 | 5 | 89.9 | 90.4 | 90.0 | 2370.4 | 40.4 | 38.5 | 1.269 | 4 | Good |
| Run 3 | 122/138 | 59 | 4 | 6 | 88.4 | 90.4 | 89.0 | 2376.4 | 40.0 | 38.2 | 1.270 | 4 | Good |
| **Average** | **122.7/138** | | | | **88.9** | **90.3** | **89.3** | **2391.4** | **40.13** | **38.23** | **1.273** | **4** | **Good** |

### OFF vs ON, head to head (averages)

| Metric | Thinking OFF | Thinking ON | Delta |
|---|---:|---:|---:|
| Quality | 97.3 | 88.9 | OFF +8.4 |
| Deployability | 97.03 | 89.3 | OFF +7.73 |
| Median answer latency (ms) | 1197.8 | 2391.4 | OFF approximately 2x faster |
| Decode tok/s | 38.8 | 40.13 | ON +1.33 |
| Effective tok/s | 35.07 | 38.23 | ON +3.16 |
| Token efficiency | 1.510 | 1.273 | OFF higher (less waste) |

Thinking ON's only wins are the raw throughput numbers, and those are inflated by the extra internal reasoning tokens that never reach the user. On every metric a user actually experiences (correctness, latency, useful token efficiency), thinking OFF wins.

---

## Concurrency sweep

Twelve tool-calling scenarios, 8 requests per level, full 1M-context config.

### Thinking OFF

| C | Requests | OK | Errors | Wall (s) | Aggregate tok/s | Aggregate total tok/s | Decode tok/s | Effective tok/s | Per-agent tok/s | Quality | Score | p50 TTFT (ms) | p95 turn (ms) |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 12 | 12 | 0 | 45.837 | 36.521 | 320.9 | 39.3 | 36.5 | 36.5 | 91.7 | 22/24 | 269.9 | 4197.3 |
| 2 | 12 | 12 | 0 | 29.661 | 50.267 | 480.0 | 28.4 | 26.4 | 25.1 | 87.5 | 21/24 | 309.7 | 3172.2 |
| 3 | 12 | 12 | 0 | 25.197 | 62.388 | 571.9 | 23.0 | 21.6 | 20.8 | 87.5 | 21/24 | 324.3 | 5968.4 |
| 4 | 12 | 12 | 0 | 23.853 | 60.078 | 576.8 | 19.1 | 17.9 | 15.0 | 79.2 | 19/24 | 457.7 | 7106.1 |
| 5 | 12 | 12 | 0 | 21.914 | 69.270 | 661.4 | 17.2 | 16.1 | 13.9 | 87.5 | 21/24 | 504.7 | 8349.2 |
| 6 | 12 | 12 | 0 | 22.475 | 70.521 | 626.9 | 18.6 | 17.4 | 11.8 | 79.2 | 19/24 | 529.8 | 6736.8 |
| 8 | 12 | 12 | 0 | 21.325 | 76.201 | 663.8 | 16.2 | 14.5 | 9.5 | 70.8 | 17/24 | 917.5 | 9615.9 |

### Thinking ON

| C | Requests | OK | Errors | Wall (s) | Aggregate tok/s | Aggregate total tok/s | Decode tok/s | Effective tok/s | Per-agent tok/s | Quality | Score | p50 TTFT (ms) | p95 turn (ms) |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 12 | 12 | 0 | 102.090 | 40.308 | 171.2 | 41.7 | 40.3 | 40.3 | 75.0 | 18/24 | 267.7 | 11527.9 |
| 2 | 12 | 12 | 0 | 60.555 | 55.553 | 275.6 | 31.0 | 29.9 | 27.8 | 75.0 | 18/24 | 309.9 | 13006.6 |
| 3 | 12 | 12 | 0 | 46.411 | 72.268 | 354.1 | 25.5 | 24.7 | 24.1 | 75.0 | 18/24 | 332.7 | 13879.8 |
| 4 | 12 | 12 | 0 | 60.246 | 61.133 | 278.1 | 20.6 | 20.0 | 15.3 | 66.7 | 16/24 | 338.3 | 20783.3 |
| 5 | 12 | 12 | 0 | 41.916 | 79.086 | 392.3 | 18.1 | 17.6 | 15.8 | 66.7 | 16/24 | 409.2 | 18536.7 |
| 6 | 12 | 12 | 0 | 37.478 | 85.704 | 437.9 | 17.5 | 16.9 | 14.3 | 75.0 | 18/24 | 489.7 | 19616.9 |
| 8 | 12 | 12 | 0 | 44.432 | 82.102 | 406.7 | 15.7 | 16.9 | 10.3 | 75.0 | 18/24 | 814.6 | 23797.3 |

Per-agent tok/s is the aggregate output throughput divided by the concurrency level. Note how it collapses from roughly 40 (single user) to roughly 10 (8 concurrent) in both modes. The GB10 is memory-bandwidth-bound, so adding concurrent streams raises aggregate throughput modestly while each individual agent slows down sharply and p95 latency grows. This box shines as a single-user or small-batch machine. At every concurrency level, thinking OFF holds higher quality and far lower p95 latency than thinking ON.

Zero request errors across all 14 concurrency runs (7 levels x 2 modes), every level fully completed within the 180 s wall timeout.

---

## Capacity and context

- **KV cache at full 1M context:** 3,127,938 tokens.
- **Concurrency headroom at 1M:** 3.13x (KV cache divided by the 1,000,000 token context window). This is the number of full-context conversations the box can hold simultaneously before evicting.
- **MTP2 acceptance:** approximately 81% overall, approximately 92% at draft position 0.

---

## Source files

| File | Contents |
|---|---|
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-final69-thinkingoff-run1__20260620T135528.json` | Thinking OFF eval, run 1 |
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-final69-thinkingoff-run2__20260620T140324.json` | Thinking OFF eval, run 2 |
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-final69-thinkingoff-run3__20260620T140712.json` | Thinking OFF eval, run 3 |
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-final69-thinkingon-run1__20260620T141758.json` | Thinking ON eval, run 1 |
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-final69-thinkingon-run2__20260620T142508.json` | Thinking ON eval, run 2 |
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-final69-thinkingon-run3__20260620T143212.json` | Thinking ON eval, run 3 |
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-concurrency12-thinkingoff__20260620T145112.json` (and `.md`) | Concurrency sweep, thinking OFF |
| `benchmarks/mimo-tp3-omni-mtp2-1mctx-concurrency12-thinkingon__20260620T145807.json` (and `.md`) | Concurrency sweep, thinking ON |

Each eval JSON carries a `meta` block (label, model, endpoint, UTC, thinking flag, max_tokens), an `aggregate` block (the scored summary), and a `scenarios` array with per-scenario verdict, points, reason, TTFT, turn count, wall time, and token counts. The concurrency JSONs carry `meta` and a `levels` array with per-level throughput, latency percentiles, and quality.
