---
type: concept
title: llama.cpp vs Ollama on an older Pascal datacenter GPU
description: How llama.cpp compares to Ollama for throughput and KV-cache efficiency on a previous-generation (Pascal) datacenter GPU.
tags:
- llama-cpp
- ollama
- benchmarks
- gpu
- moe
- kv-cache
status: draft
resource:
created: 2026-09-27T19:48:56Z
updated: 2026-09-27T19:48:56Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:48:56Z
verified: []
stale_after: 2028-09-26T19:48:56Z
sources:
- id: 2026-09-27-llama-cpp-vs-ollama-benchmarks
  resource: private:/sources/ai/models/2026-09-27-llama-cpp-vs-ollama-benchmarks.md
relations: []
superseded_by:
---

# llama.cpp vs Ollama on an older Pascal datacenter GPU

Running an upstream `llama.cpp` build alongside Ollama on the same GPU is a reasonable way to get
finer control (custom quantized KV cache, newer model support) without disturbing an existing
Ollama setup. On an older Pascal-generation datacenter GPU (e.g. a Tesla P40, 24 GB, no native
FP8/BF16 support), the comparison has a few traps that are easy to misread.

## How it works

Build `llama.cpp` from upstream source as a second, independent inference engine — nothing about
an existing Ollama install needs to change; they can run side by side (see *Pitfalls* for the one
thing that does need managing). Benchmark with `llama-bench` (llama.cpp's built-in standardized
tool), typically:

```bash
llama-bench -m <model.gguf> -p 512 -n 128 -r 3 -ngl 999 -ctk q8_0 -ctv q8_0
```

- `-p 512 -n 128 -r 3` — 512-token prompt processing, 128-token generation, averaged over 3 runs.
- `-ngl 999` — full GPU offload.
- `-ctk q8_0 -ctv q8_0` — quantized KV cache (critical for fitting long contexts on an older card,
  see below).

## When to use it / trade-offs

**The dense-vs-MoE trap.** Model family tags on a hosting registry (e.g. two size variants of the
same model family) can look like the same architecture at different parameter counts when file
sizes are close, but turn out to be genuinely different architectures — one dense (all parameters
active every token), the other a Mixture-of-Experts (MoE, only a small fraction of parameters
active per token). Comparing a dense model's throughput against an MoE model's and calling it
"llama.cpp vs Ollama speed" is comparing two different architectures, not two engines. The only
fair comparison is same-architecture vs. same-architecture (e.g. MoE-vs-MoE). In one such
same-architecture comparison on a Tesla P40, llama.cpp and Ollama landed within about 1% of each
other on generation speed — any large gap you measure is far more likely to be an architecture
mismatch than an engine difference.

**KV-cache efficiency can differ meaningfully between Ollama's fork and upstream llama.cpp for
the same nominal architecture** — always measure empirically rather than trusting either project's
documented rate for a given model. See
[Sizing context windows by measured VRAM/KV-cache cost](context-window-vram-sizing.md) for the
general measurement method; the finding specific to this comparison is below.

## Pitfalls

- **Quantize the KV cache, not just the weights.** `--cache-type-k q8_0 --cache-type-v q8_0`
  (quantizing the KV cache itself) was necessary to get anywhere close to usable context sizes on
  a 24 GB Pascal card — the default F16 KV cache roughly doubles memory cost per token versus
  q8_0. A model on Q4-class weight quantization still needs the *full-precision* KV cache math
  unless the server explicitly quantizes the KV cache too.
- **An MoE model with the same weight footprint as a dense one can have a dramatically smaller KV
  cache**, not because of the parameter-activation savings (that affects compute, not KV cache
  size) but because of a shorter attention window or a mixed attention pattern the MoE variant
  happens to use. Don't assume a smaller *active* parameter count implies a smaller KV cache —
  check the attention pattern (full vs. sliding-window vs. hybrid), as described in
  [Sizing context windows by measured VRAM/KV-cache cost](context-window-vram-sizing.md).
- **Two inference engines sharing one GPU is a real contention risk**, not a bug in either engine.
  A default multi-minute "keep model warm" timeout (Ollama's `keep_alive`, by default 5 minutes)
  can leave a model resident in VRAM long enough that a second engine's load attempt on the same
  GPU fails or crash-loops, purely from lack of free VRAM. If benchmarking two engines
  back-to-back on the same card, explicitly unload the first engine's model rather than waiting
  for its idle timer — e.g. for Ollama:
  ```bash
  curl http://127.0.0.1:11434/api/generate -d '{"model":"<model>","keep_alive":0}'
  ```
- **Quantization-Aware Training (QAT) checkpoints matter for low-bit inference quality.** A model
  released specifically with QAT calibration for a given quantization level can noticeably
  outperform a naive post-training quantization at the same bit-width — and going to *higher*
  precision than the published QAT file can sometimes hurt accuracy, if the checkpoint wasn't
  trained for that precision. Check whether a model's quantized GGUF was QAT-calibrated before
  assuming "higher bit-width = better."

## Related

- [Sizing context windows by measured VRAM/KV-cache cost](context-window-vram-sizing.md) — the
  general method for measuring safe context ceilings, referenced throughout this comparison.

## Sources

- [Original benchmark notes (private)](../../../../../sources/ai/models/2026-09-27-llama-cpp-vs-ollama-benchmarks.md)
