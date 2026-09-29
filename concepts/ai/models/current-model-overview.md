---
type: concept
title: Current LLM coding leaderboard
description: A refreshed top-100 coding leaderboard combining BenchLM coding scores with OpenRouter token pricing.
tags: []
status: draft
resource:
created: 2026-09-29T21:15:15Z
updated: 2026-09-29T21:20:53Z
generated:
  by: hermes/gpt-6-luna
  at: 2026-09-29T21:20:53Z
verified: []
stale_after: 2028-09-28T21:15:15Z
sources:
- id: benchlm-coding-api
  resource: https://benchlm.ai/api/data/leaderboard?category=coding&limit=100
- id: benchlm-data-license
  resource: https://benchlm.ai/data
- id: openrouter-model-catalog
  resource: https://openrouter.ai/api/v1/models
relations: []
superseded_by:
---

# Current LLM coding leaderboard

A live-generated snapshot of BenchLM's coding-ranked lane, joined conservatively to OpenRouter's public API catalog for current API pricing. It is a shortlist, not a universal quality or value verdict; evaluate finalists on your own workload.

## Snapshot and ranked lane

- Generated/fetched: 2026-09-29T21:20:52Z (UTC); BenchLM payload last updated: 2026-09-29; BenchLM HTTP fetched: Tue, 29 Sep 2026 21:20:52 GMT; OpenRouter HTTP fetched: Tue, 29 Sep 2026 21:20:52 GMT.
- BenchLM snapshot: `2026-09-29-3413f907dee2b747`; methodology: `bench-align-v5.7-2026-09-24`; license: **CC BY-NC 4.0**. Attribution: **Data from BenchLM.ai**. Non-commercial use only under that license; commercial use requires permission from BenchLM.
- This table uses the rank order and `categoryScores.coding` values from the requested coding endpoint. It includes the endpoint's evidence labels, including supported/reported/estimated rows; it is not a verified-only lane. Scores and ranks are source-provided, not recalculated here.
- BenchLM's aggregate methodology and underlying benchmark set can change between snapshots. Do not read small score/rank changes as capability changes without checking the methodology version and source evidence.

## Leaderboard

| Rank | Model (provider) | BenchLM coding | Evidence | Input $/1M | Output $/1M | Context | BenchLM source | OpenRouter |
|---:|---|---:|---|---:|---:|---:|---|---|
| 1 | [Claude Sonnet 5.5](https://openrouter.ai/models/anthropic/claude-sonnet-5.5) (Anthropic) | 85.12 | supported | $2 | $10 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-sonnet-5.5) |
| 2 | [Claude Opus 5.5](https://openrouter.ai/models/anthropic/claude-opus-5.5) (Anthropic) | 83.60 | supported | $4 | $20 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-opus-5.5) |
| 3 | [Claude Fable 5.1](https://openrouter.ai/models/anthropic/claude-fable-5.1) (Anthropic) | 79.99 | supported | $10 | $50 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-fable-5.1) |
| 4 | [GPT-6 Astra](https://openrouter.ai/models/openai/gpt-6-astra) (OpenAI) | 74.18 | supported | $10 | $50 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-6-astra) |
| 5 | [Claude Fable 5](https://openrouter.ai/models/anthropic/claude-fable-5) (Anthropic) | 73.10 | supported | $10 | $50 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-fable-5) |
| 6 | [Claude Opus 5](https://openrouter.ai/models/anthropic/claude-opus-5) (Anthropic) | 72.18 | supported | $5 | $25 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-opus-5) |
| 7 | [GPT-5.6 Sol](https://openrouter.ai/models/openai/gpt-5.6-sol) (OpenAI) | 70.53 | supported | $2 | $10 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.6-sol) |
| 8 | [GPT-6.1 Sol](https://openrouter.ai/models/openai/gpt-6.1-sol) (OpenAI) | 66.97 | supported | $2 | $10 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-6.1-sol) |
| 9 | [MiMo-V2.6-Pro](https://openrouter.ai/models/xiaomi/mimo-v2.6-pro) (Xiaomi) | 66.29 | supported | $0.435 | $0.87 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/xiaomi/mimo-v2.6-pro) |
| 10 | [GPT-5.6 Terra](https://openrouter.ai/models/openai/gpt-5.6-terra) (OpenAI) | 64.41 | supported | $2 | $12 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.6-terra) |
| 11 | [Gemini 3.8 Flash](https://openrouter.ai/models/google/gemini-3.8-flash) (Google) | 64.16 | supported | $0.75 | $3.75 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemini-3.8-flash) |
| 12 | [GPT-5.6 Luna](https://openrouter.ai/models/openai/gpt-5.6-luna) (OpenAI) | 63.30 | supported | $0.2 | $1.2 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.6-luna) |
| 13 | [GPT-6 Sol](https://openrouter.ai/models/openai/gpt-6-sol) (OpenAI) | 62.99 | supported | $2 | $10 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-6-sol) |
| 14 | [GPT-5.5](https://openrouter.ai/models/openai/gpt-5.5) (OpenAI) | 62.83 | supported | $5 | $30 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.5) |
| 15 | [Claude Opus 4.8](https://openrouter.ai/models/anthropic/claude-opus-4.8) (Anthropic) | 62.37 | supported | $5 | $25 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-opus-4.8) |
| 16 | [Grok 4.6](https://openrouter.ai/models/x-ai/grok-4.6) (xAI) | 61.62 | supported | $2 | $6 | 500,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/x-ai/grok-4.6) |
| 17 | [Kimi K3](https://openrouter.ai/models/moonshotai/kimi-k3) (Moonshot AI) | 61.59 | supported | $3 | $15 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/moonshotai/kimi-k3) |
| 18 | [Step 5 Preview](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (StepFun) | 61.44 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 19 | [Hy4 preview](https://openrouter.ai/models/tencent/hy4-preview) (Tencent) | 60.82 | supported | $0.834 | $2.501 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/tencent/hy4-preview) |
| 20 | [Claude Sonnet 5](https://openrouter.ai/models/anthropic/claude-sonnet-5) (Anthropic) | 59.96 | supported | $2 | $10 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-sonnet-5) |
| 21 | [Gemini 3.7 Flash](https://openrouter.ai/models/google/gemini-3.7-flash) (Google) | 59.48 | supported | $0.75 | $3.75 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemini-3.7-flash) |
| 22 | [Claude Opus 4.7](https://openrouter.ai/models/anthropic/claude-opus-4.7) (Anthropic) | 58.10 | supported | $5 | $25 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-opus-4.7) |
| 23 | [Grok 4.5](https://openrouter.ai/models/x-ai/grok-4.5) (xAI) | 58.07 | supported | $2 | $6 | 500,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/x-ai/grok-4.5) |
| 24 | [Claude Opus 4.7 (Adaptive)](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Anthropic) | 57.89 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 25 | [GLM-5.3](https://openrouter.ai/models/z-ai/glm-5.3) (Z.AI) | 56.84 | supported | $1.4 | $4.4 | 1,310,720 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-5.3) |
| 26 | [GLM-5.2](https://openrouter.ai/models/z-ai/glm-5.2) (Z.AI) | 56.29 | supported | $0.2807 | $3.99 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-5.2) |
| 27 | [Muse Spark 1.1](https://openrouter.ai/models/meta/muse-spark-1.1) (Meta) | 55.66 | supported | $1.25 | $4.25 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/meta/muse-spark-1.1) |
| 28 | [Qwen3.8 Max](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Alibaba) | 55.60 | supported | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 29 | [GPT-5.3 Codex](https://openrouter.ai/models/openai/gpt-5.3-codex) (OpenAI) | 55.50 | supported | $1.75 | $14 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.3-codex) |
| 30 | [Muse Spark 1.2](https://openrouter.ai/models/meta/muse-spark-1.2) (Meta) | 55.25 | supported | $1.25 | $4.25 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/meta/muse-spark-1.2) |
| 31 | [Qwen3.8-Flash-Next](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Alibaba) | 55.16 | supported | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 32 | [Ornith-1.5-397B](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Ornith AI) | 55.10 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 33 | [Claude Opus 4.6 (Adaptive)](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Anthropic) | 54.16 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 34 | [Gemini 3.6 Flash](https://openrouter.ai/models/google/gemini-3.6-flash) (Google) | 53.78 | supported | $0.75 | $3.75 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemini-3.6-flash) |
| 35 | [Muse Spark](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Meta) | 53.26 | supported | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 36 | [GLM-5.3-Flash](https://openrouter.ai/models/z-ai/glm-5.3-flash) (Z.AI) | 52.54 | supported | $0.15 | $0.5 | 1,310,720 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-5.3-flash) |
| 37 | [Gemini 3.5 Flash](https://openrouter.ai/models/google/gemini-3.5-flash) (Google) | 52.43 | supported | $1.5 | $9 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemini-3.5-flash) |
| 38 | [GPT-6 Luna](https://openrouter.ai/models/openai/gpt-6-luna) (OpenAI) | 51.37 | supported | $0.1 | $0.5 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-6-luna) |
| 39 | [Hy3](https://openrouter.ai/models/tencent/hy3) (Tencent) | 50.99 | supported | $0.0825 | $0.33 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/tencent/hy3) |
| 40 | [GLM-5.1](https://openrouter.ai/models/z-ai/glm-5.1) (Z.AI) | 50.80 | supported | $0.9646 | $3.0316 | 204,800 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-5.1) |
| 41 | [GPT-5.4](https://openrouter.ai/models/openai/gpt-5.4) (OpenAI) | 49.83 | supported | $2.5 | $15 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.4) |
| 42 | [Claude Opus 4.6](https://openrouter.ai/models/anthropic/claude-opus-4.6) (Anthropic) | 49.68 | supported | $5 | $25 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-opus-4.6) |
| 43 | [DeepSeek V4.1 Flash](https://openrouter.ai/models/deepseek/deepseek-v4.1-flash) (DeepSeek) | 49.63 | estimated | $0.3 | $1.2 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/deepseek/deepseek-v4.1-flash) |
| 44 | [MiMo-V2.6-Flash](https://openrouter.ai/models/xiaomi/mimo-v2.6-flash) (Xiaomi) | 49.61 | supported | $0.14 | $0.28 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/xiaomi/mimo-v2.6-flash) |
| 45 | [DeepSeek V4 Pro 0813](https://openrouter.ai/models/deepseek/deepseek-v4-pro-0813) (DeepSeek) | 49.36 | supported | $0.3942 | $3.49 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/deepseek/deepseek-v4-pro-0813) |
| 46 | [Qwen3.8-27B](https://openrouter.ai/models/qwen/qwen3.8-27b) (Alibaba) | 48.88 | supported | $0.0249 | $4.4 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.8-27b) |
| 47 | [Qwen 3.6 Max (preview)](https://openrouter.ai/models/qwen/qwen3.6-max-preview) (Alibaba) | 47.07 | supported | $1.027 | $6.162 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.6-max-preview) |
| 48 | [Claude Sonnet 4.6](https://openrouter.ai/models/anthropic/claude-sonnet-4.6) (Anthropic) | 47.04 | supported | $3 | $15 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-sonnet-4.6) |
| 49 | [Kimi K2.6](https://openrouter.ai/models/moonshotai/kimi-k2.6) (Moonshot AI) | 46.36 | supported | $0.65 | $3.41 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/moonshotai/kimi-k2.6) |
| 50 | [Qwen3.7 Max](https://openrouter.ai/models/qwen/qwen3.7-max) (Alibaba) | 45.80 | supported | $1.475 | $4.425 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.7-max) |
| 51 | [MiMo-V2.5-Pro](https://openrouter.ai/models/xiaomi/mimo-v2.5-pro) (Xiaomi) | 45.54 | supported | $0.435 | $0.87 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/xiaomi/mimo-v2.5-pro) |
| 52 | [GPT-5.2-Codex](https://openrouter.ai/models/openai/gpt-5.2-codex) (OpenAI) | 45.23 | supported | $1.75 | $14 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.2-codex) |
| 53 | [Gemini 3 Pro](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Google) | 44.98 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 54 | [Kimi K2.7 Code](https://openrouter.ai/models/moonshotai/kimi-k2.7-code) (Moonshot AI) | 44.45 | supported | $0.6562 | $3.3 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/moonshotai/kimi-k2.7-code) |
| 55 | [Qwen3.7 Plus](https://openrouter.ai/models/qwen/qwen3.7-plus) (Alibaba) | 43.06 | estimated | $0.32 | $1.28 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.7-plus) |
| 56 | [Claude Opus 4.5](https://openrouter.ai/models/anthropic/claude-opus-4.5) (Anthropic) | 42.56 | estimated | $5 | $25 | 200,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/anthropic/claude-opus-4.5) |
| 57 | [Qwen3.6 Plus](https://openrouter.ai/models/qwen/qwen3.6-plus) (Alibaba) | 42.54 | supported | $0.325 | $1.95 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.6-plus) |
| 58 | [Gemini 3.1 Pro](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Google) | 42.44 | supported | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 59 | [Quasar 438B](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Multiverse Computing) | 40.72 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 60 | [Apodex 1.1](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Apodex) | 40.44 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 61 | [Inkling-Small](https://openrouter.ai/models/thinkingmachines/inkling-small) (Thinking Machines Lab) | 40.18 | supported | $0.45 | $1.2 | 524,288 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/thinkingmachines/inkling-small) |
| 62 | [GPT-5.2](https://openrouter.ai/models/openai/gpt-5.2) (OpenAI) | 39.70 | supported | $1.75 | $14 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.2) |
| 63 | [GLM-4.7](https://openrouter.ai/models/z-ai/glm-4.7) (Z.AI) | 39.41 | supported | $0.6 | $2.2 | 204,800 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-4.7) |
| 64 | [GLM-5](https://openrouter.ai/models/z-ai/glm-5) (Z.AI) | 39.17 | estimated | $0.6 | $1.92 | 204,800 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-5) |
| 65 | [GPT-5.1](https://openrouter.ai/models/openai/gpt-5.1) (OpenAI) | 38.86 | supported | $1.25 | $10 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.1) |
| 66 | [Kimi K2.5](https://openrouter.ai/models/moonshotai/kimi-k2.5) (Moonshot AI) | 38.82 | estimated | $0.45 | $2.25 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/moonshotai/kimi-k2.5) |
| 67 | [MiniMax M3](https://openrouter.ai/models/minimax/minimax-m3) (MiniMax) | 38.81 | supported | $0.3 | $1.2 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/minimax/minimax-m3) |
| 68 | [Gemini 3.5 Flash-Lite](https://openrouter.ai/models/google/gemini-3.5-flash-lite) (Google) | 38.48 | supported | $0.3 | $2.5 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemini-3.5-flash-lite) |
| 69 | [MiMo-V2-Pro](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Xiaomi) | 38.18 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 70 | [MiMo-V2.5](https://openrouter.ai/models/xiaomi/mimo-v2.5) (Xiaomi) | 37.76 | supported | $0.14 | $0.28 | 1,050,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/xiaomi/mimo-v2.5) |
| 71 | [Qwen3.6-27B](https://openrouter.ai/models/qwen/qwen3.6-27b) (Alibaba) | 37.26 | supported | $0.32 | $3.2 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.6-27b) |
| 72 | [GPT-5.4 mini](https://openrouter.ai/models/openai/gpt-5.4-mini) (OpenAI) | 36.92 | supported | $0.75 | $4.5 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.4-mini) |
| 73 | [Inkling](https://openrouter.ai/models/thinkingmachines/inkling) (Thinking Machines Lab) | 36.56 | supported | $1 | $4.05 | 524,288 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/thinkingmachines/inkling) |
| 74 | [Gemini 3 Flash](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Google) | 36.38 | supported | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 75 | [Qwen3.6-35B-A3B](https://openrouter.ai/models/qwen/qwen3.6-35b-a3b) (Alibaba) | 36.25 | estimated | $0.15 | $1 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.6-35b-a3b) |
| 76 | [Muse Glimmer 30B](https://openrouter.ai/models/meta/muse-glimmer-30b) (Meta) | 36.02 | estimated | $0.3 | $1.2 | 131,072 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/meta/muse-glimmer-30b) |
| 77 | [MiniMax M2.7](https://openrouter.ai/models/minimax/minimax-m2.7) (MiniMax) | 36.00 | supported | $0.21 | $0.84 | 204,800 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/minimax/minimax-m2.7) |
| 78 | [MiniMax M2.5](https://openrouter.ai/models/minimax/minimax-m2.5) (MiniMax) | 35.75 | estimated | $0.27 | $1.08 | 204,800 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/minimax/minimax-m2.5) |
| 79 | [Qwen3.5-122B-A10B](https://openrouter.ai/models/qwen/qwen3.5-122b-a10b) (Alibaba) | 35.64 | supported | $0.26 | $2.08 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.5-122b-a10b) |
| 80 | [Gemma 4 31B](https://openrouter.ai/models/google/gemma-4-31b-it) (Google) | 35.53 | supported | $0.09 | $0.34 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemma-4-31b-it) |
| 81 | [GPT-5 (medium)](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (OpenAI) | 34.19 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 82 | [Ling 3.0 Flash](https://openrouter.ai/models/inclusionai/ling-3.0-flash) (InclusionAI) | 34.07 | supported | $0.021 | $0.063 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/inclusionai/ling-3.0-flash) |
| 83 | [GLM-5V-Turbo](https://openrouter.ai/models/z-ai/glm-5v-turbo) (Z.AI) | 34.02 | estimated | $1.2 | $4 | 202,752 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-5v-turbo) |
| 84 | [Gemma 4 26B A4B](https://openrouter.ai/models/google/gemma-4-26b-a4b-it) (Google) | 33.95 | supported | $0.09 | $0.3 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemma-4-26b-a4b-it) |
| 85 | [GPT-5.1-Codex](https://openrouter.ai/models/openai/gpt-5.1-codex) (OpenAI) | 32.71 | estimated | $1.25 | $10 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.1-codex) |
| 86 | [DeepSeek V3.2 (Thinking)](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (DeepSeek) | 32.42 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 87 | [Qwen3.5-27B](https://openrouter.ai/models/qwen/qwen3.5-27b) (Alibaba) | 32.29 | estimated | $0.195 | $1.56 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/qwen/qwen3.5-27b) |
| 88 | [MiMo-V2-Flash](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Xiaomi) | 32.12 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 89 | [DeepSeek V3.2](https://openrouter.ai/models/deepseek/deepseek-v3.2) (DeepSeek) | 32.09 | estimated | $0.28 | $0.42 | 163,840 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/deepseek/deepseek-v3.2) |
| 90 | [Step 3.7 Flash](https://openrouter.ai/models/stepfun/step-3.7-flash) (StepFun) | 32.04 | estimated | $0.2 | $1.15 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/stepfun/step-3.7-flash) |
| 91 | [GPT-5.4 nano](https://openrouter.ai/models/openai/gpt-5.4-nano) (OpenAI) | 31.19 | supported | $0.2 | $1.25 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5.4-nano) |
| 92 | [Hy3 Preview](https://openrouter.ai/models/tencent/hy3-preview) (Tencent) | 30.49 | estimated | $0.18 | $0.6 | 262,144 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/tencent/hy3-preview) |
| 93 | [o1](https://openrouter.ai/models/openai/o1) (OpenAI) | 30.40 | estimated | $15 | $60 | 200,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/o1) |
| 94 | [GPT-5 mini](https://openrouter.ai/models/openai/gpt-5-mini) (OpenAI) | 29.88 | estimated | $0.25 | $2 | 400,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/openai/gpt-5-mini) |
| 95 | [Grok 4.3](https://openrouter.ai/models/x-ai/grok-4.3) (xAI) | 29.45 | supported | $1.25 | $2.5 | 1,000,000 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/x-ai/grok-4.3) |
| 96 | [Laguna M.1](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (Poolside) | 28.68 | supported | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 97 | [Gemini 3.1 Flash-Lite](https://openrouter.ai/models/google/gemini-3.1-flash-lite) (Google) | 27.66 | estimated | $0.25 | $1.5 | 1,048,576 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/google/gemini-3.1-flash-lite) |
| 98 | [K-Exaone](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (LG AI Research) | 27.53 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |
| 99 | [GLM-4.6](https://openrouter.ai/models/z-ai/glm-4.6) (Z.AI) | 27.39 | estimated | $0.43 | $1.75 | 204,800 | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | [model page](https://openrouter.ai/models/z-ai/glm-4.6) |
| 100 | [Grok 4.1 Fast (Reasoning)](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) (xAI) | 27.22 | estimated | — | — | — | [ranked data](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) | — |

Prices are OpenRouter's `pricing.prompt` and `pricing.completion` USD per token multiplied by 1,000,000, i.e. standard input/output rates; cache, batch, tool, and long-context tier prices are excluded. Context is OpenRouter's listed token limit, not a guarantee of effective context. Exact normalized model-name matches with a unique OpenRouter catalog entry only; uncertain or ambiguous matches remain unlinked and unpriced (—). OpenRouter catalog name matching can miss aliases or distinct product variants; verify before purchase.

## Interpretation and caveats

- BenchLM ranks its coding aggregate, not a universal measure of coding-agent performance. Its methodology version and each row's evidence status are shown above/in the table.
- Scores from SWE-bench and human-preference arenas are intentionally not mixed into this ranking: their tasks, harnesses, populations, and scales differ. A separate result belongs here only when the exact model variant and benchmark setup are independently matched and verifiable.
- Per-1M prices are provider catalog rates as observed at refresh time, not a quote; provider routing, discounts, caches, context tiers, and output/reasoning token accounting can change effective cost.
- Missing data is shown as — rather than inferred. Benchmark inclusion does not imply OpenRouter availability, and an OpenRouter match does not imply benchmark equivalence.

## Changelog

- 2026-09-29: Initial generated snapshot (100 ranked models).

## Sources and license

- [BenchLM machine-readable coding leaderboard](https://benchlm.ai/api/data/leaderboard?category=coding&limit=100) — ranked rows, scores, evidence labels, snapshot and methodology metadata.
- [BenchLM dataset and methodology/licensing overview](https://benchlm.ai/data) — data coverage, ranking lanes, update date and attribution. **Data from BenchLM.ai**, licensed **CC BY-NC 4.0**; commercial use is not permitted without a separate license.
- [BenchLM methodology](https://benchlm.ai/methodology) — score construction and verification approach.
- [Creative Commons Attribution-NonCommercial 4.0](https://creativecommons.org/licenses/by-nc/4.0/).
- [OpenRouter model catalog API](https://openrouter.ai/api/v1/models) — model identifiers, listed context lengths and per-token input/output pricing.
- [How to compare coding benchmarks](coding-benchmark-comparison.md) — explanation of benchmark differences and interpretation.
