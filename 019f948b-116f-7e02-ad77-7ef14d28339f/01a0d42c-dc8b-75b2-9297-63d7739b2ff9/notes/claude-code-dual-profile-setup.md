---
title: Claude Code Dual Profile Setup (Worldly + Direct)
kind: fact
tags:
- claude-code
- worldly
- profile
- infra
pinned: true
tier: semantic
type: Fact
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-25T16:59:25Z
---
Three Claude Code wrappers coexist on this machine:

| Command | Profile | Backend |
|---|---|---|
| `claude` | Worldly gateway | LiteLLM via `https://llm.development.worldly.io` |
| `claude-worldly` | Worldly gateway (explicit) | Same as above |
| `claude-direct` | Default | Standard Anthropic OAuth |

## How it works

### `claude` (Worldly gateway)
- PATH shim at `~/.worldly-claude/bin/claude` (prepended to PATH in `.zshrc`)
- Calls `~/.local/bin/claude-worldly` which sets `CLAUDE_CONFIG_DIR=~/.claude-modes/cli/worldly-gateway`
- Process wrapper `~/.local/bin/worldly-claude-process-wrapper` injects gateway env vars and edge header
- Profile: `~/.claude-modes/cli/worldly-gateway/`

### `claude-direct` (standard Anthropic)
- Script at `~/.local/bin/claude-direct`
- Explicitly **unsets** all Worldly gateway env vars before launching
- Profile: `~/.claude/` (default)

### Profile sync (2026-09-24)
The two profiles share the same hooks, plugins, commands, agents, and skills:

- **settings.json** — worldly profile merged with all default hooks (ai-memory × 9, GSD × 5, RTK × 1), status line, plugins config, TUI, theme
- **commands/** → symlinked from `~/.claude/commands/`
- **agents/** → symlinked from `~/.claude/agents/`
- **skills/** → symlinked from `~/.claude/skills/`
- **CLAUDE.md, RTK.md** → symlinked
- **plugins/** → all subdirs symlinked (cache, marketplaces, code-review, data, synced, + manifests)

The only difference between profiles: the worldly profile has gateway-specific `apiKeyHelper`, `env`, `model`, and `modelPicker` keys.