---
type: concept
title: 'LLM wiki: a beginner’s guide'
description: A simple explanation of an AI-maintained knowledge base, its portable file format, and how to keep it trustworthy.
tags:
- llm
- knowledge-base
- wiki
status: draft
resource:
created: 2026-09-27T18:42:08Z
updated: 2026-09-27T18:42:08Z
generated:
  by: hermes/gpt-6-sol
  at: 2026-09-27T18:42:08Z
verified: []
stale_after: 2028-09-26T18:42:08Z
sources:
- id: karpathy-llm-wiki
  resource: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- id: karpathy-capture
  resource: private:/sources/ai/tooling/2026-09-27-llm-wiki-pattern.md
- id: google-okf-intro
  resource: https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing/?hl=en
- id: okf-v02-spec
  resource: https://github.com/GoogleCloudPlatform/open-knowledge-format/blob/main/SPEC.md
- id: google-okf-capture
  resource: private:/sources/ai/tooling/2026-09-27-open-knowledge-format-introduction.md
- id: rohit-llm-wiki-v2
  resource: https://gist.github.com/rohitg00/2067ab416f7bbe447c1977edaaa681e2
- id: rohit-capture
  resource: private:/sources/ai/tooling/2026-09-27-llm-wiki-v2-pattern.md
relations: []
superseded_by:
---

# LLM wiki: a beginner’s guide

**In one sentence:** An LLM wiki is a collection of linked notes that an AI helps keep up to date using saved source material. Instead of piecing together the same answer from scratch every time, it builds knowledge you can reuse and improve.[^karpathy-llm-wiki]

Imagine a careful librarian. You bring an article; the librarian keeps the original, writes a short note about it, connects that note to related notes, and revises old notes when the article changes what you know. You decide what is worth keeping and review important changes. The AI does the filing and cross-checking.

## The three pieces

1. **Sources:** Original articles, documents, and observations. Keep them as evidence; do not quietly rewrite them.
2. **Wiki pages:** Short explanations of people, topics, decisions, and how things connect. These *can* change when new evidence arrives.
3. **Rules (schema):** Instructions for the AI: where files go, how to cite evidence, when to update a page, and how to handle disagreement.[^karpathy-llm-wiki]

## What happens in practice?

**Add a source → update the relevant pages → link them → ask questions → check their health.** For example, a new article about a tool may improve an existing tool page, not require another nearly identical page. When sources disagree, record both claims and ask for review rather than silently choosing one. A periodic check finds broken links, stale facts, and pages nobody connects to.[^karpathy-llm-wiki]

This differs from searching original documents only when a question arrives: the useful summary and connections remain in the wiki for the next question. Search can still help find sources and pages; it is not the enemy of a wiki.[^karpathy-llm-wiki]

## How can a person or another AI read it?

**Open Knowledge Format (OKF)** is a way to package knowledge as ordinary Markdown files with small YAML information blocks at the top (for example, a page's `type`, `title`, and `description`). The files can live in Git, open in a text editor, and move between tools without a special database. An index helps readers find pages and a log records changes. The linked Google article introduces OKF **v0.1**; a later v0.2 specification adds more explicit provenance, verification, and freshness fields.[^google-okf-intro][^okf-v02-spec]

## How do we keep it useful and safe?

- **Show the trail:** Cite the source of a claim and distinguish an AI-written note from something a person has checked.
- **Watch for age and change:** Flag old or contradicted claims; keep superseded knowledge for history rather than pretending it never existed.
- **Protect privacy:** Remove secrets and decide what is private before publishing anything.
- **Review regularly:** Check links, contradictions, missing topics, and stale pages. Let the human decide consequential changes.[^rohit-llm-wiki-v2]

The v2 essay also suggests richer relationships, automated checks, and better search for large wikis. These are **optional upgrades**, not requirements for starting. It proposes numerical confidence scores; an OKF reader can instead judge trust from cited sources, who verified a page, and when it was last checked.[^rohit-llm-wiki-v2]

## Go deeper

1. [Karpathy — LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): the original idea and the ingest/query/lint workflow.
2. [Google Cloud — Introducing the Open Knowledge Format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing/?hl=en): why the files are portable and readable across tools (v0.1 introduction).
3. [rohitg00 — LLM Wiki v2](https://gist.github.com/rohitg00/2067ab416f7bbe447c1977edaaa681e2): optional ideas for freshness, trust, automation, and scale.

For implementers: [the later OKF v0.2 specification](https://github.com/GoogleCloudPlatform/open-knowledge-format/blob/main/SPEC.md).

[^karpathy-llm-wiki]: Karpathy, *LLM Wiki*.
[^google-okf-intro]: McVeety and Hormati, *Introducing the Open Knowledge Format*.
[^okf-v02-spec]: GoogleCloudPlatform, *Open Knowledge Format v0.2 specification*.
[^rohit-llm-wiki-v2]: rohitg00, *LLM Wiki v2*.
