---
type: concept
title: Comparing LLMs by coding benchmark
description: What common coding benchmarks (SWE-bench, LiveCodeBench-style aggregates, arena ratings) actually measure and how to read a price/performance leaderboard.
tags:
- llm
- benchmark
- swe-bench
status: draft
resource:
created: 2026-09-27T17:15:28Z
updated: 2026-09-27T17:38:00Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T17:15:28Z
verified: []
stale_after: 2028-09-26T17:15:28Z
sources:
- {id: 2026-09-27-current-model-overview, resource: 'private:/sources/ai/models/2026-09-27-current-model-overview.md'}
- {id: benchlm-coding, resource: 'https://benchlm.ai/coding'}
- {id: swe-bench-verified, resource: 'https://llm-stats.com/benchmarks/swe-bench-verified'}
- {id: lmarena-leaderboard, resource: 'https://lmarena.ai/leaderboard'}
- {id: openrouter-rankings, resource: 'https://openrouter.ai/rankings'}
relations: []
superseded_by:
---

# Comparing LLMs by coding benchmark

Public LLM leaderboards rank models by different benchmarks that measure different things. Reading
a leaderboard usefully means knowing what each column actually tests, not just which model is
"#1" today — rankings from this space change weekly and any specific numbers go stale almost
immediately.

## How it works

Common categories of coding/agentic benchmark seen on leaderboards:

- **Aggregate coding scores** (e.g. a weighted blend of an agentic-fix benchmark and a
  competitive-programming benchmark): meant to approximate "how good is this model at realistic
  coding tasks end to end," but the exact weighting varies by provider and changes over time —
  treat the number as an ordinal ranking signal, not an absolute score comparable across sites.
- **SWE-bench-style benchmarks:** measure whether a model can resolve real, verified GitHub issues
  end to end (read an issue, patch a real codebase, pass the existing test suite). Closer to actual
  agentic coding work than pure code-generation benchmarks, but scores vary a lot by scaffolding
  (the harness/tool access given to the model), so cross-provider comparisons on this number alone
  can be misleading unless the harness is held constant.
- **Arena / human-preference ratings:** crowd-sourced pairwise comparisons voted on by users,
  aggregated into an Elo-style rating. Captures subjective quality and instruction-following, not
  narrow task-completion accuracy — a model can rank high here and mediocre on SWE-bench, or vice
  versa.
- **Price per 1M tokens (input/output):** the input price matters most for large-context/RAG-heavy
  workloads; output price dominates for long, reasoning-heavy responses. Compare price *and*
  benchmark together — a much cheaper model that's only slightly worse is often the better
  default, and a "top of leaderboard" model is frequently 5-10x the cost of one a few points behind.

## When to use it / trade-offs

Use a leaderboard to shortlist 2-4 candidates for a specific workload, then verify with your own
task samples — none of these benchmarks are perfect proxies for a specific use case (e.g. a model
strong on SWE-bench-style issue resolution is not automatically the best choice for a chat
assistant, and vice versa). Re-check pricing and rank at the point you actually need to choose;
don't rely on a snapshot more than a few weeks old.

## Pitfalls

- **Self-reported vs. independently verified scores.** Some numbers on leaderboards are reported
  by the model's own provider and later revised (sometimes lowered) once an independent site
  reproduces the benchmark. Prefer numbers from a site that verifies runs itself, and be skeptical
  of numbers that only ever appear on the provider's own materials.
- **Scaffolding-dependent benchmarks.** SWE-bench-style and other agentic benchmarks are run with
  a specific tool/harness setup; the same underlying model can score very differently under a
  better or worse harness. A benchmark table rarely tells you which harness was used.
- **Context length ≠ usable context.** A large advertised context window doesn't mean quality holds
  up near that limit — some models degrade well before their nominal maximum. No leaderboard
  column captures this; test it directly for long-context use cases.
- **Weekly churn.** Rankings and prices from these sources move frequently (new model releases,
  price cuts, benchmark re-verification). Don't hard-code a "best model" decision into
  documentation — link to the live leaderboard instead of copying a table of current numbers.

## Related

- [Sizing context windows by measured VRAM/KV-cache cost](context-window-vram-sizing.md) — for
  self-hosted models, the benchmark score is only half the picture; whether it fits your hardware
  at a usable context length is the other half.

## References

Check the live listings rather than a point-in-time copy:

- [BenchLM coding leaderboard](https://benchlm.ai/coding) — aggregate coding score
- [SWE-bench Verified](https://llm-stats.com/benchmarks/swe-bench-verified)
- [LMArena leaderboard](https://lmarena.ai/leaderboard) — human-preference ratings
- [OpenRouter rankings](https://openrouter.ai/rankings) — usage and per-model pricing
