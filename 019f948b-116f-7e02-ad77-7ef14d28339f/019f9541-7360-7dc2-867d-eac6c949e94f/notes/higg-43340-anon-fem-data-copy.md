---
tags:
- anon-scripts
- higg-43340
- elasticsearch
- cleanup-script
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-08-12T16:40:22Z
---
# HIGG-43340: Copying FEM 2025 prod data to Dev test accounts — session recap

## Reference
Jira: [HIGG-43340](https://worldly.atlassian.net/browse/HIGG-43340) — "BE - Copy FEM Data from
Production". Task, assigned to Wellington Lorindo, status In Progress as of this session. Victor
Ferraz's comment on the ticket pointed at `ts/scripts/anon/anon-download.ts` +
`ts/scripts/anon/anon-upload.ts` as the tool to use.

## Task
HIGG-43340: copy FEM 2025 data from 2 production accounts to 2 Development test accounts using
`ts/scripts/anon/anon-download.ts` + `anon-upload.ts`:
- `5a2ab9ecb38a6604a0a52e78` → `6a6d0be87b3f32d084c90a42` ("Rainha")
- `5a848d6f4aea5704a43ecdc0` → `6a6d0d487b3f32d084c90a43`

Both targets are **pre-existing** Development test accounts (not created by the anon tooling).

## Key technical learnings (reusable beyond this ticket)

**anon-upload/anon-purge specific-IDs mode has real gaps**, discovered while trying to target a
pre-existing account instead of letting the tool mint a brand-new one:
- `anon-upload.ts` never writes back the anon IDs it generates to `/tmp/{targetAccountId}/`. Those
  files always hold the original source-env IDs.
- Because of that, `anon-purge.ts`'s specific-IDs discovery (reads those same files) finds 0
  matches in the destination env — it's broken for genuine cross-environment specific-IDs runs,
  not just for a workaround.
- `anon-purge.ts`'s network-mode path hard-aborts unless the given root account itself has
  `anonymized: true` — so it can't be pointed at a pre-existing, non-anonymized target either.
- **Workaround that works**: anon-upload's specific-IDs mode doesn't actually parse the content of
  `--assessmentIds`/`--assessmentIdsFile` — `resolveMode()` only checks the *flags are present* to
  pick the code path. So you can reuse already-downloaded shareNetwork data by renaming
  `/tmp/{sourceAccountId}/` → `/tmp/{targetAccountId}/` and editing `metadata.json`'s `mode` field
  to `"specific-ids"` (+ adding `targetAccountId`), then run upload with
  `--targetAccountId=<id> --assessmentIds=x` (dummy value, just needs to be non-empty). No
  re-download needed.
- Wrote a one-off cleanup script for this exact gap: `ts/local/higg-43340-cleanup.ts` (untracked,
  don't commit). Rediscovers anon accounts/assessments via `DataServices.sharesES` rooted at the
  (non-anonymized) pre-existing target account, plus a second pass for VerificationBody accounts +
  anon-to-anon shares preserved during anonymization (invisible from pass 1 since the share
  recipient differs). Deletes with the same `anonymized`/`@anon.com` safety checks `anon-purge.ts`
  uses. Never touches the target account itself.

**ES scroll/search-after pagination bug pattern** (bit us hard — confirmed via log, ~2100 sequential
round trips for a check that should've taken ~21): raw ES search requests default to **10 hits per
page** if you don't set an explicit `size`. Always set it explicitly on any hand-rolled
`searchGenerator`/scroll query — mirrors `Elasticsearch.getAllIds()`'s own explicit `size: 500`
(`ts/es/elasticsearch.ts:576-587`).

**Redundant-delete false-failure bug**: `UserCouch.destroyEntity` (`ts/couch/user-couch.ts:106-121`)
already cascades to ES + PG internally on success. Calling `userES.deleteEntity()` again afterward
404s against an already-deleted doc, gets caught, and wrongly marks a fully-successful delete as
failed. Same bug class the `deleteAssessments` comment in `anon-purge.ts` explicitly calls out
(HIGG-40541) — but `anon-purge.ts`'s own `deleteUsers` still has it (not yet fixed upstream; fixed
in our local copy).

**`PartialAssessment` (`ts/types/api.ts:1153`) has no `_id` field** even though ES always returns
one — a real typing gap, not something requestable via the `source` array param. When you need a
typed `_id` from an ES scan, use `searchGenerator()` directly (`ESHit<T>._id` is properly typed)
instead of `getPartialAssessments`.

**rtk (user's global token-saving CLI proxy) can fabricate false success output.** `npx tsc`
crashed (real V8 native crash, OOM-looking) but rtk printed `TypeScript: No errors found` anyway —
confirmed by reading the raw tee log at `~/Library/Application Support/rtk/tee/*.log`, which showed
the actual crash stack. To get a trustworthy tsc result, bypass rtk: call
`./node_modules/.bin/tsc --noEmit -p tsconfig.json` directly with output redirected to a file and
read that file directly.

## Outcome as of this session

- **Run 1** (target `6a6d0be87b3f32d084c90a42`, "Rainha"): ✅ succeeded cleanly. 232 accounts, 752
  assessments, 752/752 shares — matches expected counts. Some non-fatal noise (`invalid
  sipfacilitytype` warnings from assessments missing that survey answer — logged, doesn't throw;
  ~50 per-assessment `Error bulk indexing in ES` during creation — caught per-item, didn't block
  the run).
- **Run 2** (target `6a6d0d487b3f32d084c90a43`): ❌ failed. Crashed mid-way through the
  share-creation loop with a fatal `Error bulk indexing in ES` — only **129 of ~1521** expected
  shares were created before it died. No "Specific-IDs upload complete" summary was ever logged.
  Root cause not recoverable from the log: `Elasticsearch.search()`'s catch
  (`ts/es/elasticsearch.ts:273`) only logs the generic `ResponseError` message, swallowing
  `e.meta.body` where the real ES rejection reason would be. Also worth knowing: `anon-upload.ts`'s
  `main()` calls `process.exit(0)` unconditionally even after this kind of fatal error, so **exit
  code 0 does NOT mean success** for this tool.

## Open follow-up
Target `6a6d0d487b3f32d084c90a43` needs: purge via `ts/local/higg-43340-cleanup.ts
--targetAccountId=6a6d0d487b3f32d084c90a43 --dryrun=false`, then re-run the upload from the
still-intact `/tmp/6a6d0d487b3f32d084c90a43/` data. Not yet done as of this session. Progress
should be reported back on [HIGG-43340](https://worldly.atlassian.net/browse/HIGG-43340).
