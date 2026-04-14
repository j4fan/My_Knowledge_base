---
title: Context Compaction
type: concept
date: 2026-04-14
sources: 1
---

## Definition

Context compaction is the family of strategies [Claude Code](../entities/claude-code.md) uses to keep the active conversation history within the model's token budget. When the conversation grows too large, one or more compaction strategies reduce its size — either by deleting old content, summarizing it in-place, or archiving it to an external store — before the next API call is made.

## Strategies

Nine strategies exist, organized by scope and trigger:

### Per-turn (proactive), run in this order each turn

| # | Strategy | Scope | Model | Reversible | Control |
|---|----------|-------|-------|------------|---------|
| 1 | **Tool Result Budget** | Truncates oversized individual tool results | none | No | always on |
| 2 | **History Snip** | Drops oldest messages from the start of history | none | No | `HISTORY_SNIP` flag |
| 3 | **Microcompact** | Summarizes old tool results in-place | Haiku | No | always on |
| 3a | **Cached Microcompact** | Same as Microcompact but edits prompt cache in-place | Haiku | No | `CACHED_MICROCOMPACT` (ant-only) |
| 4 | **Context Collapse** | Archives whole conversation segments to a commit-log; projected on read | context agent (*marble_origami*) | Soft — segments kept in collapse store | `CONTEXT_COLLAPSE` flag |
| 5 | **Autocompact** | Full conversation replaced by a summary when token count ≥ 167k | main model | No — resets turn tracking, emits `compact_boundary` | `autoCompactEnabled`; disable with `DISABLE_AUTO_COMPACT` |

### After a 413 prompt-too-long API error

| # | Strategy | Scope | Model | Reversible | Control |
|---|----------|-------|-------|------------|---------|
| 6 | **Collapse Drain** | Commits all staged Context Collapse entries at once | none | Soft | `CONTEXT_COLLAPSE` flag; guard: `transition !== 'collapse_drain_retry'` |
| 7 | **Reactive Compact** | Full conversation summary as fallback | main model | No | `REACTIVE_COMPACT` flag |

### User-initiated

| # | Strategy | Scope | Model | Reversible | Control |
|---|----------|-------|-------|------------|---------|
| 8 | **Session Memory Compact** | Uses persisted session memory facts instead of a live summary | fast model | No | experimental; `sessionMemoryCompact.ts` |
| 9 | **Manual Compact** | Full conversation summary on demand | main model | No | `/compact` command; disable with `DISABLE_COMPACT=1` |

## Key Distinctions

- **Destructive vs. soft:** History Snip, Microcompact, Autocompact, and Reactive Compact are irreversible — old content is gone from the session. Context Collapse is softer: archived segments are retained in the collapse store, just hidden from the API.
- **Scope:** Microcompact and Tool Result Budget work at the individual-tool-result level. All other strategies operate on the whole conversation.
- **Trigger:** Proactive strategies (Snip, Microcompact, Autocompact, Collapse) run every turn before the API call. Reactive Compact runs only after the API rejects with a 413.

## Circuit Breaker

`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`. After 3 consecutive autocompact failures, the system stops retrying and surfaces the error to the user. Introduced after a 2026-03-10 BigQuery analysis found 1,279 sessions with 50+ consecutive failures (up to 3,272 per session), wasting ≈250,000 API calls per day fleet-wide.

## Related Concepts

- [Token Thresholds](token-thresholds.md)

## Sources

- [Claude Code Memory & Compaction Strategies](../sources/claude-code-memory.md)
