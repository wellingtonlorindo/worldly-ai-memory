---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.6.1
  at: 2026-10-08T22:22:12Z
---
# ai-memory upgrade & backup setup

How the ai-memory MCP lib is installed, upgraded, and backed up on this machine (created 2026-10-08).

## Install / upgrade

- ai-memory is a **Docker-based install**: the `ai-memory` command at `~/.local/bin/ai-memory` is a thin bash wrapper around image `akitaonrails/ai-memory:latest`.
- Data lives in a **bind-mounted** host dir (NOT a named volume): real path `~/.local/share/ai-memory-data`, bind-mounted at `/data` in the container.
- Upgrade command: `ai-memory upgrade` — pulls latest image, self-updates wrapper, refreshes hooks + `settings.json`, then needs a container **recreate** to take effect.
- Ran 2.0.1 → 2.6.1 on 2026-10-08. Recreate script kept at `~/.cache/ai-memory/recreate-ai-memory.sh` (stop/rm/run with the bind mount).

## Backup

- Memory source of truth = the **wiki git repo** at `~/.local/share/ai-memory-data/wiki/.git` (markdown pages). The `db/memory.sqlite` (1.3GB) is an *index*, rebuildable from wiki via `ai-memory reindex` — do NOT git the sqlite.
- wiki repo is pushed to private remote: `git@github.com:wellingtonlorindo/worldly-ai-memory.git` (branch `master`).
- ai-memory **auto-commits** wiki on writes, but does NOT auto-push. Sync is done by a launchd job.

## Auto-sync (hourly push)

- launchd agent: `~/Library/LaunchAgents/com.wellingtonlorindo.ai-memory-sync.plist` → runs `/Users/wellingtonlorindo/.local/share/ai-memory/sync-to-remote.sh` every 3600s + `RunAtLoad`.
- The script commits dirty wiki files, then pushes `master` to the remote. Uses `GIT_SSH_COMMAND` forcing the **passphraseless** `~[REDACTED:credential_path] key (SSH config's `~[REDACTED:credential_path] + `UseKeychain yes` don't work for background jobs).
- Log at `~/.local/share/ai-memory/sync.log`.

## Critical gotcha — macOS TCC blocks launchd from ~/Documents

The data originally lived at `~/Documents/code/worldly/ai-memory-data`. **launchd-spawned processes cannot read `~/Documents`** (macOS TCC Documents protection). Symptom: `fatal: Unable to read current working directory: Operation not permitted` from git under launchd, while the same command works in an interactive terminal (terminal has the TCC grant; launchd does not). No git flag/`cd`/cwd trick bypasses it.

Fix: moved data to **`~/.local/share/ai-memory-data`** (unprotected). A symlink keeps the old path working:
`~/Documents/code/worldly/ai-memory-data -> ~/.local/share/ai-memory-data`.

Lesson: **any background job (launchd/cron) that must read/write a repo should NOT live under `~/Documents`.**

## Related
- [[account-service-tickets-overview]] for project context (separate concern — memory is NOT in the api.higg.org repo).
