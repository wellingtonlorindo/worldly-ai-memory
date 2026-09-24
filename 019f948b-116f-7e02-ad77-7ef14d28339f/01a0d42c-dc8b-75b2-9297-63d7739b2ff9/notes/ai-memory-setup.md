---
title: AI Memory System Configuration
kind: fact
tags:
- ai-memory
- memory
- infra
pinned: true
tier: semantic
type: Fact
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-24T18:42:03Z
---
A persistent, cross-agent memory system (ai-memory by akitaonrails) is running locally.

## Endpoints

- **Server:** `http://127.0.0.1:49374` (Docker container `ai-memory`)
- **Data directory:** `/Users/wellingtonlorindo/Documents/code/worldly/ai-memory-data`
- **Wiki:** `/Users/wellingtonlorindo/Documents/code/worldly/ai-memory-data/wiki/`
- **GitHub:** https://github.com/akitaonrails/ai-memory

## Docker

```bash
docker ps --filter name=ai-memory
```

Image: `akitaonrails/ai-memory:latest`, bound to `127.0.0.1:49374`.

## CLI

The `ai-memory` binary is at `~/.local/bin/ai-memory`. Key commands:

| Command | Purpose |
|---|---|
| `ai-memory search <query>` | Full-text search the wiki |
| `ai-memory write-page --path <p> --body <b>` | Write/update a wiki page |
| `ai-memory read-page <query>` | Read a specific page |
| `ai-memory status` | Runtime counts, paths, version |
| `ai-memory backup` | Snapshot wiki/db into tarball |
| `ai-memory commit` | Stage + commit wiki under git |

Pages are indexed with FTS5 and tagged with workspace/project auto-detection.

## Claude Code Integration

Hooks are installed in `~/.claude/settings.json` covering: SessionStart, PostToolUse, PreToolUse, UserPromptSubmit, PreCompact, Stop, SessionEnd, SubagentStart, SubagentStop. All post to `http://127.0.0.1:49374` via `AI_MEMORY_HOOK_URL`.

These hooks are also present in the worldly gateway profile at `~/.claude-modes/cli/worldly-gateway/settings.json` (merged from default profile on 2026-09-24).