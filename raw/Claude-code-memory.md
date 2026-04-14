---
  All Compaction Strategies in Claude Code

  #: 1
  Strategy: History Snip
  Trigger: Token count exceeds a threshold; runs every turn
  Scope: Drops oldest messages from the start of history, keeps a protected tail
  Model used: none — pure deletion
  Reversible?: No
  Flag / control: HISTORY_SNIP feature flag
  ────────────────────────────────────────
  #: 2
  Strategy: Microcompact
  Trigger: Runs every turn before the API call
  Scope: Summarises old tool results within the existing message list (e.g. large Read/Bash outputs)
  Model used: fast model (Haiku)
  Reversible?: No
  Flag / control: always on
  ────────────────────────────────────────
  #: 3
  Strategy: Cached Microcompact
  Trigger: Same as microcompact
  Scope: Same as microcompact, but edits the prompt cache in-place instead of re-sending changed content
  Model used: fast model (Haiku)
  Reversible?: No
  Flag / control: CACHED_MICROCOMPACT (ant-only)
  ────────────────────────────────────────
  #: 4
  Strategy: Context Collapse
  Trigger: Proactive, per-turn
  Scope: Archives whole conversation segments into summaries stored in a separate commit-log. View is projected on read;

    REPL retains full history
  Model used: context agent ("marble_origami")
  Reversible?: Soft — archived segments kept in collapse store
  Flag / control: CONTEXT_COLLAPSE feature flag
  ────────────────────────────────────────
  #: 5
  Strategy: Autocompact (proactive)
  Trigger: Token count ≥ autocompact threshold (~167k on 200k model)
  Scope: Full conversation summary — replaces the entire history with a summary message
  Model used: main model
  Reversible?: No — resets turn tracking, emits compact_boundary
  Flag / control: autoCompactEnabled setting; disable with DISABLE_AUTO_COMPACT
  ────────────────────────────────────────
  #: 6
  Strategy: Reactive Compact
  Trigger: Triggered reactively after a 413 prompt-too-long API error
  Scope: Full conversation summary, same as autocompact
  Model used: main model
  Reversible?: No
  Flag / control: REACTIVE_COMPACT feature flag
  ────────────────────────────────────────
  #: 7
  Strategy: Collapse Drain
  Trigger: 413 received, before reactive compact is attempted
  Scope: Commits all staged context-collapses at once to free space
  Model used: none — pure projection
  Reversible?: Soft
  Flag / control: CONTEXT_COLLAPSE feature flag; guard: transition !== 'collapse_drain_retry'
  ────────────────────────────────────────
  #: 8
  Strategy: Session Memory Compact
  Trigger: Attempted first inside the /compact command path, and optionally proactively
  Scope: Compacts using persisted session memory (extracted facts/context from prior turns) instead of a live summary
  Model used: fast model
  Reversible?: No
  Flag / control: experiment; sessionMemoryCompact.ts
  ────────────────────────────────────────
  #: 9
  Strategy: Manual Compact (/compact)
  Trigger: User-initiated via /compact slash command
  Scope: Full conversation summary, same as autocompact
  Model used: main model
  Reversible?: No
  Flag / control: always available unless DISABLE_COMPACT=1

  ---
  How they layer — order of execution each turn

  Every loop iteration:
    1. Tool Result Budget  →  truncate oversized individual results
    2. History Snip        →  drop oldest messages (HISTORY_SNIP)
    3. Microcompact        →  summarise old tool results in-place
         └─ Cached variant →  edit prompt cache instead of resending (CACHED_MICROCOMPACT)
    4. Context Collapse    →  project archived segments (CONTEXT_COLLAPSE)
    5. Autocompact         →  full summary if threshold crossed

  After API returns 413:
    6. Collapse Drain      →  commit all staged collapses first (cheap)
    7. Reactive Compact    →  full summary as fallback

  User action:
    9. Manual /compact     →  tries Session Memory first, falls back to full summary

  Key distinctions

  - Destructive vs. soft: Snip, microcompact, autocompact, and reactive compact are irreversible — old content is gone
  from the session. Context collapse is softer: the archived segments are kept in the collapse store, just hidden from
  the API.
  - Scope: Microcompact works at the individual-tool-result level. Everything else works at the whole-conversation
  level.
  - Trigger: Proactive (snip, microcompact, autocompact, collapse) run every turn before the API call. Reactive compact
  runs only after the API actually rejects the request with a 413.
  - Circuit breaker: After 3 consecutive autocompact failures, the system stops retrying to avoid burning ~250k API
  calls/day (observed in BQ data from 2026-03-10).



  ---
  All Token Thresholds in the System

  The model context window is 200,000 tokens by default. The system carves it up into zones. Working backwards from the
  top:

  200,000  ── raw context window
  ─ 20,000  ── reserved for summary output (MAX_OUTPUT_TOKENS_FOR_SUMMARY)
  ─────────
  180,000  ── effectiveContextWindow
  ─ 13,000  ── autocompact buffer (AUTOCOMPACT_BUFFER_TOKENS)
  ─────────
  167,000  ── autocompact threshold  ← compact fires here (proactive)
  ─  7,000  ── (implicit gap)
  ─────────
  160,000  ── warning threshold  (autocompact threshold − 20,000)
             = "context is getting full" UI indicator
  ─────────
  160,000  ── error threshold  (same buffer: autocompact threshold − 20,000)
             = red UI indicator
  ─────────
  177,000  ── blocking limit  (effectiveContextWindow − 3,000 MANUAL buffer)
             = hard stop when auto-compact is OFF

  ---
  Each threshold explained

  Threshold: effectiveContextWindow
  Value (200k model): ~180,000
  Constant: contextWindow − MAX_OUTPUT_TOKENS_FOR_SUMMARY
  Purpose: Usable space after reserving room for the compact summary output (p99.99 summary = 17,387 tokens → reserved
    20k)
  ────────────────────────────────────────
  Threshold: autocompact threshold
  Value (200k model): ~167,000
  Constant: effectiveContextWindow − AUTOCOMPACT_BUFFER_TOKENS (13,000)
  Purpose: Fires the proactive full compaction. The 13k buffer ensures the compaction itself (which generates output)
  has
    enough headroom to succeed
  ────────────────────────────────────────
  Threshold: warning threshold
  Value (200k model): ~147,000
  Constant: autocompact threshold − WARNING_THRESHOLD_BUFFER_TOKENS (20,000)
  Purpose: UI turns yellow — "context window is getting full"
  ────────────────────────────────────────
  Threshold: error threshold
  Value (200k model): ~147,000
  Constant: autocompact threshold − ERROR_THRESHOLD_BUFFER_TOKENS (20,000)
  Purpose: UI turns red (same buffer, but separate flag in case they diverge)
  ────────────────────────────────────────
  Threshold: blocking limit
  Value (200k model): ~177,000
  Constant: effectiveContextWindow − MANUAL_COMPACT_BUFFER_TOKENS (3,000)
  Purpose: Hard stop when auto-compact is disabled. Reserves 3k so the user can still run /compact manually without
    hitting a 413

  ---
  Why each buffer is sized the way it is

  MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000
  Based on real data: p99.99 of compact summary outputs was 17,387 tokens. They rounded up to 20k. Without this
  reservation, the compaction call itself could hit the context limit mid-generation.

  AUTOCOMPACT_BUFFER_TOKENS = 13,000
  This gap between the effective window and the autocompact trigger exists so that by the time the loop detects "over
  threshold" and fires compaction, the conversation hasn't already overflowed. The compaction model call needs a few
  thousand tokens of breathing room.

  WARNING_THRESHOLD_BUFFER_TOKENS = 20,000
  Warning fires 20k tokens before the autocompact threshold. This gives the UI time to show the indicator before
  anything disruptive happens.

  MANUAL_COMPACT_BUFFER_TOKENS = 3,000
  Only used when DISABLE_AUTO_COMPACT=1. In that mode there's no automatic compaction, so the system synthetically stops
   the query at 177k to ensure the user always has enough room to manually run /compact themselves.

  ---
  How token count is measured — tokenCountWithEstimation()

  The tricky part is that you can't just count characters — you need the API's actual token count. The function:

  1. Walks backwards through the message list to find the most recent assistant message that has a real API usage
  object.
  2. Takes the usage.input_tokens + cache_creation + cache_read + output_tokens from that response (the actual context
  size the API reported).
  3. Estimates any messages added after that response using roughTokenCountEstimationForMessages() (character length ÷
  ~4).
  4. Handles a parallel-tool-call edge case: when multiple tools fire in one turn, the streaming code emits separate
  assistant records all sharing the same message.id. It walks back to the first sibling of that ID so all interleaved
  tool_result messages between splits are counted.

  The result is context window size at this moment — not total tokens ever generated, not just output tokens. This is
  what every threshold comparison uses.

  ---
  The MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3 circuit breaker

  A BQ analysis (2026-03-10) found 1,279 sessions had 50+ consecutive autocompact failures in a single session — up to
  3,272 failures — wasting ~250k API calls/day fleet-wide. The circuit breaker stops retrying after 3 consecutive
  failures. After that, the error surfaces to the user instead of burning API budget in an unrecoverable loop.
