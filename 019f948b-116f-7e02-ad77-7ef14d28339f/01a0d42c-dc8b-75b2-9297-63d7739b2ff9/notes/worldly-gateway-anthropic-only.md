---
title: Worldly LiteLLM Gateway is Anthropic-Only
kind: fact
tags:
- worldly
- litellm
- gateway
tier: semantic
type: Fact
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-24T18:39:32Z
---
The Worldly LiteLLM gateway at `https://llm.development.worldly.io` **only proxies the Anthropic Messages API** at `/v1/messages`. It does **not** serve OpenAI-format `/v1/chat/completions` — those return HTTP 403 Forbidden.

This is critical for any tool being wired against it.

## Required headers

| Header | Source |
|---|---|
| `x-api-key` | Personal LiteLLM virtual key (from macOS Keychain `worldly-litellm-gateway`) |
| `anthropic-version` | `2023-06-01` |
| `X-Worldly-Edge-Key` | Shared ALB edge secret (from macOS Keychain `worldly-litellm-edge`) |

## Available models

`claude-haiku-4-5`, `claude-opus-5`, `claude-fable-5`, `claude-deepseek-v4-pro`, `claude-deepseek-v4-flash`, `claude-kimi-k3-preview`, `claude-glm-5-3`, `claude-glm-5-3-flash`

## Keychain entries

- Service `worldly-litellm-gateway`, account `wellington.lorindo@worldly.io` — personal key
- Service `worldly-litellm-edge`, account `wellington.lorindo@worldly.io` — ALB edge secret

## Tool implications

When configuring any tool against this gateway, always use `protocol: anthropic` (not `openai`). The Anthropic-native endpoint is the only one available.