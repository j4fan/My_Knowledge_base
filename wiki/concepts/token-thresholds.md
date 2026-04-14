---
title: Token Thresholds
type: concept
date: 2026-04-14
sources: 1
---

## Definition

Token thresholds are the specific token-count values at which Claude Code's context management system changes behavior — displaying warnings, triggering compaction, or hard-blocking new input. The thresholds are derived from the raw context window size by subtracting a set of reserved buffers.

## Threshold Map (200k model)

```
200,000  ── raw context window
─ 20,000  ── reserved for summary output (MAX_OUTPUT_TOKENS_FOR_SUMMARY)
─────────
180,000  ── effectiveContextWindow
─ 13,000  ── autocompact buffer (AUTOCOMPACT_BUFFER_TOKENS)
─────────
167,000  ── autocompact threshold   ← proactive compaction fires here
─ 20,000  ── warning/error buffer
─────────
147,000  ── warning threshold       ← UI turns yellow
147,000  ── error threshold         ← UI turns red (same value, separate flag)

177,000  ── blocking limit          ← hard stop when auto-compact is OFF
            (effectiveContextWindow − MANUAL_COMPACT_BUFFER_TOKENS 3,000)
```

## Each Threshold Explained

| Threshold | Value (200k) | Formula | Purpose |
|-----------|-------------|---------|---------|
| `effectiveContextWindow` | ≈180,000 | `contextWindow − MAX_OUTPUT_TOKENS_FOR_SUMMARY` | Usable space after reserving room for the compact summary output |
| Autocompact threshold | ≈167,000 | `effectiveContextWindow − AUTOCOMPACT_BUFFER_TOKENS (13k)` | Fires proactive full compaction |
| Warning threshold | ≈147,000 | `autocompactThreshold − WARNING_THRESHOLD_BUFFER_TOKENS (20k)` | UI turns yellow — "context window is getting full" |
| Error threshold | ≈147,000 | `autocompactThreshold − ERROR_THRESHOLD_BUFFER_TOKENS (20k)` | UI turns red (same buffer as warning, separate constant) |
| Blocking limit | ≈177,000 | `effectiveContextWindow − MANUAL_COMPACT_BUFFER_TOKENS (3k)` | Hard stop when `DISABLE_AUTO_COMPACT=1` |

## Why Each Buffer Is Sized the Way It Is

- **`MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000`** — Based on real data: p99.99 of compact summary outputs was 17,387 tokens; rounded up to 20k to ensure the compaction API call never hits the context limit mid-generation.
- **`AUTOCOMPACT_BUFFER_TOKENS = 13,000`** — Breathing room between the effective window and the autocompact trigger; ensures that by the time the loop detects "over threshold", the conversation hasn't already overflowed the model call that performs compaction.
- **`WARNING_THRESHOLD_BUFFER_TOKENS = 20,000`** — Warning fires 20k tokens before the autocompact threshold, giving the UI time to show the indicator before anything disruptive happens.
- **`MANUAL_COMPACT_BUFFER_TOKENS = 3,000`** — Only used when `DISABLE_AUTO_COMPACT=1`. Synthetically stops queries at 177k so the user always has enough room to run `/compact` manually without hitting a 413.

## How Token Count Is Measured

`tokenCountWithEstimation()` works as follows:

1. Walks backwards through the message list to find the most recent assistant message with a real API `usage` object.
2. Takes `usage.input_tokens + cache_creation + cache_read + output_tokens` from that response (the actual context size the API reported).
3. Estimates any messages added after that response using `roughTokenCountEstimationForMessages()` (character length ÷ ~4).
4. Handles a parallel-tool-call edge case: when multiple tools fire in one turn, streaming emits separate assistant records all sharing the same `message.id`; the function walks back to the first sibling to include all interleaved `tool_result` messages.

The result represents the current context window size — not total tokens ever generated, and not just output tokens.

## Related Concepts

- [Context Compaction](context-compaction.md)

## Sources

- [Claude Code Memory & Compaction Strategies](../sources/claude-code-memory.md)
