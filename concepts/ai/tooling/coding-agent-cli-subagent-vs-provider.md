---
type: concept
title: Coding-agent CLI as a sub-agent vs. as a model provider
description: Why a coding-agent CLI plugs cleanly into another agent as an invoked sub-agent, but not as a chat-completions model provider with tool-calling.
tags:
- coding-agent
- cli
- acp
- tool-calling
- model-provider
- sub-agent
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
- id: 2026-09-27-hermes-cursor-integration
  resource: private:/sources/ai/agents/2026-09-27-hermes-cursor-integration.md
relations: []
superseded_by:
---

# Coding-agent CLI as a sub-agent vs. as a model provider

Commercial coding-agent CLIs (the kind that ship their own file-edit, shell and search tools)
can be wired into another AI agent framework in two very different ways, and only one of them
tends to work reliably: **invoking the CLI as a sub-agent** that does its own thing end-to-end, or
**exposing the CLI's underlying model as a chat-completions provider** that the calling framework
drives with its own tool schema. These look similar (both talk to the same product) but have
opposite reliability profiles.

## How it works

**Sub-agent route:** the calling framework shells out to the coding-agent CLI in a headless/batch
mode (e.g. `<cli> --force 'do this task'`), lets it run its own full agentic loop with its own
tools and its own model choice, and reads back a final result (plain text, or a structured
JSON/NDJSON result object if the CLI supports it). The calling framework never sees the coding
CLI's internal tool calls — it is a black box that takes a task and returns an outcome.

**Model-provider route:** the calling framework treats the coding CLI as if it were a bare LLM
endpoint — wrapping its underlying protocol (often JSON-RPC over stdio, e.g. an Agent Client
Protocol/ACP-style interface) in an OpenAI-compatible shape, so the calling framework's *own* tool
schema and prompt drive the conversation, and the coding CLI is expected to just generate text and
tool-call JSON on request like any other model backend.

## When to use it / trade-offs

- **Sub-agent route** is the right fit whenever you want the coding CLI's own tool implementations
  (its file editor, its shell execution, its own model routing/cost tiers) and are happy to treat
  it as an opaque worker. This is usually reliable because nothing about the coding CLI's own
  identity or tools needs to be suppressed — it is allowed to be exactly what it is.
- **Model-provider route** is tempting when you want the coding CLI's *underlying model* available
  as just another selectable model in the calling framework's own agent loop, using the calling
  framework's own tools instead of the CLI's. This is the fragile direction.

## Pitfalls

**The core problem with the model-provider route: identity collision.** A coding-agent CLI is not
a bare model — it ships baked into the same process with its own system prompt and its own tool
names (e.g. its own `Shell`, `Read`, `Edit` tools). When a different framework's tool schema is
injected into that same conversation, the underlying model has *two* competing sets of tool
identities to choose from, and typically has no reliable way to tell which one it's supposed to
use in a given turn. Observed failure modes when this was tried in practice:

1. The model calls **its own** (the coding CLI's) tool name instead of the calling framework's —
   the calling framework rejects it as unknown, wastes a turn, and may fall back to answering from
   its own knowledge instead of using tools at all.
2. The model refuses outright, stating (in various phrasings) that it doesn't have access to the
   calling framework's tool-call mechanism.
3. The model answers in plain text with no tool call at all.

**Prompt engineering only gets partway there.** Mitigations like an early system-prompt preamble,
a trailing "reminder" near the end of the prompt (exploiting recency), or an explicit allow-list of
tool names were each tried and each failed in a different way — one still produced the coding
CLI's own tool name, another suppressed tool calls entirely. The combination that did work
reliably was forcing the coding CLI into an isolated, single-purpose prompt mode with no other
context injected — which defeats the point of a general-purpose provider, since it means the
calling framework can't casually multi-turn through it. If a proxy technique parses tool-call
markers out of free text instead of using a structured protocol, it hits the same wall from a
different angle: the marker's *name* is still the underlying model's own choice, not something
the proxy controls.

**A working technical bridge is not the same as a working integration.** It is entirely possible
to get provider discovery, model listing, model switching and even full streamed conversational
turns working end-to-end over such a bridge — none of that proves the tool-calling loop will be
usable. Test the actual tool loop under a realistic multi-tool prompt before investing further,
rather than treating "a chat request succeeds" as sufficient validation.

**When it might become viable:** if the coding CLI's protocol ever gains an explicit mode that
suppresses its own built-in tool identity (a "bring your own tools only" session mode), the
model-provider route stops fighting an identity collision and becomes straightforward. Until then,
prefer the sub-agent route and treat the coding CLI as a worker, not as a swappable model backend.

## Related

- Applies to any pairing of a tool-shipping coding-agent CLI with a separate agent framework that
  has its own tool schema — not specific to one product.

## Sources

- [Original investigation notes (private)](../../../../../sources/ai/agents/2026-09-27-hermes-cursor-integration.md)
