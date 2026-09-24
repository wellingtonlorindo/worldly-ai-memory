---
title: Open-Code-Review Configured for Worldly Gateway
kind: fact
tags:
- open-code-review
- ocr
- worldly
- litellm
tier: semantic
type: Fact
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-24T18:39:46Z
---
`open-code-review` (`ocr`) is configured at `~/.opencodereview/config.json` to use a custom provider named `worldly` through the Worldly LiteLLM gateway.

## Configuration

- **Protocol:** `anthropic` (critical — the gateway only serves Anthropic Messages API)
- **URL:** `https://llm.development.worldly.io/v1`
- **Default model:** `claude-deepseek-v4-flash` (fast/cheap for code review)
- **Extra headers:** `X-Worldly-Edge-Key` for ALB admission
- **API key:** personal LiteLLM virtual key (from macOS Keychain)

## Switching models

```bash
ocr config set custom_providers.worldly.model <model-id>
```

Available: `claude-haiku-4-5`, `claude-opus-5`, `claude-fable-5`, `claude-deepseek-v4-pro`, `claude-deepseek-v4-flash`, `claude-kimi-k3-preview`, `claude-glm-5-3`, `claude-glm-5-3-flash`

## Test connectivity

```bash
ocr llm test
```

## Usage

```bash
ocr review           # Review staged/unstaged diffs
ocr scan             # Scan entire files
```

## Claude Code dual-setup

Three wrappers exist:

| Command | Behavior |
|---|---|
| `claude` | Worldly gateway (LiteLLM) |
| `claude-worldly` | Worldly gateway (explicit) |
| `claude-direct` | Standard Anthropic OAuth, no gateway |

`claude-direct` is at `~/.local/bin/claude-direct` and unsets all gateway env vars before launching the real binary.