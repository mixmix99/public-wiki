---
type: concept
title: Sizing context windows by measured VRAM/KV-cache cost
description: How to empirically determine the safe max context window for a local LLM based on measured VRAM cost per token.
tags:
- llm
- vram
- context-window
- kv-cache
- quantization
status: draft
resource:
created: 2026-09-27T17:15:27Z
updated: 2026-09-27T19:04:10Z
generated:
  by: hermes/gpt-6-sol
  at: 2026-09-27T19:04:10Z
verified: []
stale_after: 2028-09-26T17:15:27Z
sources:
- id: 2026-09-27-kv-cache-vram-measurements
  resource: private:/sources/ai/models/2026-09-27-kv-cache-vram-measurements.md
relations: []
superseded_by:
---

# Sizing context windows by measured VRAM/KV-cache cost

When self-hosting an LLM, the model's advertised "native max context" is rarely the context you
can actually run — VRAM for the KV cache is the real limit. This describes how to measure the
actual VRAM cost per token empirically and derive a safe maximum context window, and why that
cost varies enormously by architecture.

## How it works

1. Load the model with a given `num_ctx` and run one inference that fills close to that context.
2. Read GPU memory used (e.g. `nvidia-smi` on NVIDIA cards) right after the inference completes.
3. Repeat at 3-4 different `num_ctx` values, spread across the range you care about.
4. The relationship is linear once you're past the model's fixed weight footprint:
   `KV cost per token ≈ (VRAM at high ctx − VRAM at low ctx) / (high ctx − low ctx)`.
5. Pick a safe max: `(usable VRAM − weight footprint − safety headroom) / KV cost per token`,
   then round down. Leave headroom (roughly 1-1.5 GB) — some inference frameworks briefly need
   scratch memory beyond the steady-state KV cache.
6. Bake the chosen value into the model's serving config once (e.g. an Ollama Modelfile's
   `PARAMETER num_ctx <value>`) so every client gets a safe default without per-request
   configuration; clients can still override it downward per request if they want a smaller,
   faster context.

## When to use it / trade-offs

Use this whenever you're running a quantized model on a GPU with fixed VRAM and the model's
"native" context (often 128K-1M+ on modern releases) would clearly overflow available memory long
before that limit. It matters most on older/smaller GPUs (e.g. a previous-generation datacenter
card with 24 GB) where headroom is tight.

The KV-cache cost per token differs by an order of magnitude depending on architecture, so the
"safe max context" for two models of similar parameter count on the same GPU can differ 4-10x:

- **Dense transformer with full multi-head attention (MHA):** every layer holds a full KV cache.
  Most expensive per token.
- **Grouped-query attention (GQA):** shares key/value heads across query heads — meaningfully
  cheaper than MHA, still scales with every attention layer.
- **Hybrid SSM/Mamba + attention:** only a fraction of blocks use attention (e.g. one in every four);
  the rest are state-space layers that carry **no KV cache at all**. Very context-efficient, but
  if the attention layers use MHA rather than GQA, the per-token cost of the attention layers alone
  can still be substantial — measure it, don't assume.
- **Sliding Window Attention (SWA):** most layers cap their attention window at a small fixed size
  (e.g. 512-1,024 tokens), so those layers' VRAM cost stops growing with total sequence length; only
  the layers with global attention keep scaling. This is typically the most VRAM-efficient
  general-purpose pattern for long context.
- **Mixture-of-Experts (MoE):** parameter count and KV-cache cost are largely independent — MoE
  reduces *compute* per token (only a subset of experts activate), not the size of the KV cache,
  which depends on the attention pattern as above. An FP8 (vs. default FP16) KV cache roughly
  halves this cost again.

As a rule of thumb: architectures that combine SSM/sliding-window attention with a low-precision
KV cache can be 3x or more cheaper per token than a dense-MHA model of similar parameter count —
worth checking before assuming a large model "won't fit."

## Pitfalls

- Don't trust the model card's "native max context" as a memory planning number — it describes
  training/positional-encoding limits, not what fits in your VRAM.
- Measure at more than 2 points if you can — some frameworks have a non-linear jump near their
  scratch-memory limits, and 2 points can hide it.
- Watch for silent CPU offload: some inference servers will quietly split a model across CPU and
  GPU memory once you exceed the safe context rather than failing outright — this looks like it
  "worked" but tanks throughput. Confirm 100% GPU residency after picking your safe max.
- A model on Q4-class quantization still needs the *full-precision* KV cache math unless the
  server explicitly quantizes the KV cache too (e.g. FP8) — don't assume KV cost scales with the
  weight quantization level.

## Related

- [Comparing LLMs by coding benchmark](coding-benchmark-comparison.md) — choosing which model is
  worth fitting in the first place.
- Applies to any self-hosted LLM serving setup (Ollama, vLLM, llama.cpp, etc.) on VRAM-constrained
  hardware.

## References

- Method validated against measurements on an NVIDIA Tesla P40 (24 GB card, ~23 GB usable) running
  three architecturally different quantized models via Ollama.

## Source captures (private)

- [Original source notes (private)](../../../../../sources/ai/models/2026-09-27-kv-cache-vram-measurements.md)
