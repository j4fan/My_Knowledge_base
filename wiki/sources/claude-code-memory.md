---
title: Claude Code Memory & Compaction Strategies
type: source-summary
date: 2026-04-14
sources: 1
---

## Summary

This document catalogs all nine compaction strategies used by Claude Code to manage conversation history within the context window. It describes how the strategies layer and execute each turn in a defined order, details the token thresholds that govern when each triggers, and explains the measurement approach used to compare live token counts against those thresholds. A circuit breaker mechanism is also described, introduced after BigQuery analysis revealed that repeated compaction failures were wasting approximately 250,000 API calls per day fleet-wide.

## Key Points

- Nine distinct compaction strategies exist, ranging from lightweight per-result summarization (Microcompact) to full-conversation replacement (Autocompact, Reactive Compact, Manual /compact).
- Strategies execute in a defined order each turn: Tool Result Budget → History Snip → Microcompact → Context Collapse → Autocompact; after a 413 error: Collapse Drain → Reactive Compact.
- The 200k token context window is partitioned into zones: effective window (≈180k), autocompact threshold (≈167k), warning/error threshold (≈147k), blocking limit (≈177k when auto-compact is disabled).
- Context Collapse is the only "soft" strategy — archived segments are stored in a collapse store and projected on read; all other strategies are irreversible.
- Token count is measured via `tokenCountWithEstimation()`: uses the last API response's reported `input_tokens + cache_creation + cache_read + output_tokens`, then adds a character-based estimate for messages sent after that response.
- A circuit breaker (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`) stops autocompact retries after 3 consecutive failures; triggered by a 2026-03-10 BQ analysis showing up to 3,272 failures per session.

## Entities Mentioned

- **Claude Code** — Anthropic's CLI for agentic coding; the system this document describes.
- **Haiku** — Fast Claude model used as the summarizer in Microcompact and Session Memory Compact.
- **marble_origami** — Internal codename for the context agent that performs Context Collapse summarization.

## Concepts Mentioned

- **Context Compaction** — The family of strategies Claude Code uses to keep conversation history within the token budget; see [context-compaction](../concepts/context-compaction.md).
- **Token Thresholds** — The specific token counts at which different compaction behaviors trigger; see [token-thresholds](../concepts/token-thresholds.md).
- **History Snip** — Drops the oldest messages from the start of conversation history; no model involved.
- **Microcompact** — Summarizes old tool results in-place using Haiku; always on.
- **Cached Microcompact** — Variant of Microcompact that edits the prompt cache in-place; ant-only flag.
- **Context Collapse** — Archives whole conversation segments into a commit-log; soft/reversible.
- **Autocompact** — Proactive full-conversation summary when the 167k threshold is crossed.
- **Reactive Compact** — Full-conversation summary triggered after a 413 prompt-too-long API error.
- **Collapse Drain** — Commits all staged Context Collapse entries before attempting Reactive Compact.
- **Session Memory Compact** — Experimental strategy using persisted session memory facts instead of a live summary.
- **Manual Compact** — User-initiated via the `/compact` slash command.
- **Circuit Breaker** — Stops autocompact retries after 3 consecutive failures to avoid unbounded API waste.

## Cross-Links

- [Context Compaction](../concepts/context-compaction.md)
- [Token Thresholds](../concepts/token-thresholds.md)
- [Claude Code](../entities/claude-code.md)
