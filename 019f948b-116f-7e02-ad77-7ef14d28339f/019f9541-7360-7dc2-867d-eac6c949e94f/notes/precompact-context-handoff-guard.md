---
tags:
- hooks
- ai-memory
- context-window
- precompact
- experiment
tier: procedural
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-28T19:05:35Z
---
# PreCompact handoff guard experiment (context-window auto-save)

## Goal
Force a save of a recap to ai-memory before Claude Code compacts/loses
context, then let work continue. Explored whether a Claude Code hook can
do this automatically.

## Claude Code hook capability findings (verified against official docs, 2026-09-28)
- `PreCompact` fires before compaction (`trigger`: `manual` | `auto` | `session_start`). **Can block** via exit code 2.
- `PostCompact` fires after compaction. Cannot block, no decision mechanism.
- Neither `PreCompact` nor `PostCompact` supports injecting `additionalContext` into the model's context per docs — they are documented as observational/block-only events.
- Only `UserPromptSubmit` can inject plain-text stdout into the model's context, and only on the next prompt — not context-threshold-triggered.
- **No hook can call an MCP tool directly** (hooks are shell scripts; MCP calls only happen through the model's own tool-use turn) **and no hook can trigger `/clear` programmatically.**
- Conclusion: there is no single native mechanism that forces "save to ai-memory -> /clear -> continue" on approaching the context limit. The blocking behavior of `PreCompact` was the only lever available, and whether its block *reason* actually reaches the model (vs. only being shown to the user in the terminal) was **unconfirmed by docs** — this experiment exists to test that empirically.

## ai-memory's relevant existing feature (this already works, no hook needed)
- `mcp__ai-memory__memory_handoff_begin` — records a cross-agent/cross-session recap (summary, open_questions, next_steps). Meant exactly for end-of-session / context-reset snapshots.
- Next session's `SessionStart` hook **auto-consumes and prepends** the handoff to context — no manual fetch needed. Single-use.
- `ai-memory handoffs --json` (CLI, no LLM call) lists open handoffs — usable from a plain shell hook to detect "did a handoff just get created" without needing MCP/LLM access from the hook itself.

## What was implemented (project: api.higg.org, personal/local only)
Files (untracked, personal experiment):
- `.claude/hooks/precompact-handoff-guard.sh` — `PreCompact` hook. On `trigger=auto|manual`, checks `ai-memory handoffs --json` handoff count against a baseline captured at first fire. If unchanged, blocks (exit 2) with a message telling the model to call `memory_handoff_begin`. Capped at 3 attempts per session, then lets compaction through unguarded (safety valve against deadlock). Skips `trigger=session_start` (recovery compaction, nothing new to save).
- `.claude/hooks/postcompact-log.sh` — `PostCompact` hook, just logs completion for lifecycle visibility.
- Both wired into `.claude/settings.local.json` under `hooks.PreCompact` / `hooks.PostCompact` (gitignored, local-only — not shared with the team).
- All activity logged to `.claude/hooks/logs/precompact-guard.log` (gitignored via repo-wide `*.log` rule) — tail it live to see: `PreCompact fired` → `BLOCKING attempt N/3` → either `HANDOFF DETECTED -> ALLOW` (guard worked, model reacted to the block) or `GAVE UP after 3 attempts` (guard didn't work, block reason didn't reach the model) → `COMPACTION COMPLETE`.

## How to check whether this actually works
```bash
tail -f .claude/hooks/logs/precompact-guard.log
```
Watch during a real long session. `HANDOFF DETECTED` outcome = PreCompact block reasons DO reach the model and this pattern is viable to formalize. `GAVE UP` outcome = they don't, and the fallback plan is either (a) a soft standing rule telling the model to proactively call `memory_handoff_begin` at natural checkpoints (no hook), or (b) an external supervisor loop (outside the session) that polls and drives save+restart itself.

## Related, not part of this experiment
- `.claude/skills/git-worktree/` — worktree creation skill (node_modules/libxl/env/docs symlinking). Unrelated to this guard; mentioned here only because it was built in the same session.
