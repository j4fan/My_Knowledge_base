---
title: Claude Code
type: entity
date: 2026-04-14
sources: 1
---

## Description

Claude Code is Anthropic's official CLI tool for agentic software engineering. It runs as an interactive agent in the terminal, reads and edits files, executes shell commands, and holds multi-turn conversations with the user to complete coding tasks.

## Context Management

Claude Code uses an internal compaction system to keep conversation history within the model's 200k-token context window. Nine distinct strategies handle this automatically, ranging from lightweight per-result summarization to full-conversation replacement. See [Context Compaction](../concepts/context-compaction.md) and [Token Thresholds](../concepts/token-thresholds.md) for details.

## Sources

- [Claude Code Memory & Compaction Strategies](../sources/claude-code-memory.md)
