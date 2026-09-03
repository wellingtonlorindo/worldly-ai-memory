---
tags:
- todo
- cleanup
- ai-memory
tier: working
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-03T16:55:23Z
---
# Delete ai-memory upgrade backup tarballs

After the 2026-09-03 ai-memory upgrade (1.18.0 → 2.0.1) went smoothly and stayed stable, delete these two safety-net backups:

- `/Users/wellingtonlorindo/Documents/code/worldly/ai-memory-backups/ai-memory-backup-20260903-135241.tar.gz` (14M, taken before the upgrade, on host)
- `/data/backups/ai-memory-backup-okf-v0.2-20260903-165308.tar.gz` (13.3M, inside the ai-memory container's data dir — the tool's own pre-migration backup for the OKF v0.2 wiki format migration; host path: `/Users/wellingtonlorindo/Documents/code/worldly/ai-memory-data/backups/ai-memory-backup-okf-v0.2-20260903-165308.tar.gz`)

**Why:** Both were precautionary before an upgrade/migration that already completed and was verified (page/session/observation counts checked via `ai-memory status` and `memory_status`). No longer needed once confidence in the upgrade holds.

**How to apply:** `rm` both files once satisfied nothing regressed. Not pinned — safe to let this note itself decay/resolve after cleanup.
