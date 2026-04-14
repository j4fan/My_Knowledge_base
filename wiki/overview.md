---
title: Knowledge Base Overview
type: overview
date: 2026-04-14
sources: 0
---

# Knowledge Base Overview

This is a personal LLM-maintained knowledge base, populated incrementally as
source documents are ingested from `raw/`.

## What This Wiki Covers

### Claude Code Internals

How Anthropic's Claude Code CLI manages conversation history within the model's context window. Topics include the nine compaction strategies (History Snip, Microcompact, Context Collapse, Autocompact, Reactive Compact, and others), the token threshold system that governs when each triggers, the circuit breaker that prevents unbounded API waste, and the `tokenCountWithEstimation()` measurement approach.

## How to Navigate

- [Index](index.md) — full catalog of all pages, sorted by type
- [Log](log.md) — history of all ingest, query, and lint operations
- [Sources](sources/) — one summary page per ingested raw document
- [Entities](entities/) — named people, systems, products, and organisations
- [Concepts](concepts/) — ideas, patterns, techniques, and terminology

## Current Status

| Category | Page Count |
|----------|-----------|
| Source summaries | 1 |
| Entity pages | 1 |
| Concept pages | 2 |

*(Update this table after each significant ingest session.)*
