---
tags:
- HIGG-44104
- matlib-v2
- flatfile
- deferred
pinned: true
tier: semantic
type: Decision
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-01T14:09:12Z
---
# HIGG-44104 follow-up: MatLib V2 Submit gate still blocks on stale invalid rows (deferred)

**Status:** Deferred, not scheduled. Skipped by explicit user request on 2026-09-01 — revisit only if raised again.

## Symptom this doesn't fully fix

User had a 3-row MatLib V2 blend where only row 2 was genuinely invalid (wrong Material Type). After a failed Submit, the backend's atomic-blend rule marked all 3 rows invalid. Fixing row 2 alone and re-submitting still required manually re-editing rows 1 and 3 (which were already correct) before Submit would go through.

## What was fixed (shipped, HIGG-44104, commits 3526e23 + a8fa41f)

In `ts/managers/corp-report/bulk-upload-with-flatfile/material-matlib-v2-worksheet/flatfile-upload-matlib-v2.ts`: validators only ever added invalid stamps, never cleared them once the underlying data was fixed on a later pass (except one narrow special case for the blend-pointer message). Generalized clear-and-restamp to all fields this file owns (`baseMaterial`, `materialType`, `worldlyMaterialId`, `composition`, `name`) via `clearStaleValidationMessages`, called at the top of each blend's validation pass. Also fixed a second bug: a fully-cleared blend's rows weren't written back to Flatfile if some other blend in the same batch was still invalid (`changedRecordIds` tracking added to `validateBlendRecords`).

## Why that's not the whole fix — root cause chain (per Fable consult)

1. The Flatfile "Submit data" action for this sheet has `constraints: [{ type: "hasAllValid" }, { type: "hasData" }]` (`flatfile-config-matlib-v2.ts`). Flatfile's hosted UI disables the Submit button while ANY row shows invalid — including stale marks — so the backend never even runs to clear them.
2. `processFilesInBackgrond` (`ts/managers/corp-report/bulk-upload-with-flatfile/index.ts`, ~line 526) fetches records with `filter: "valid"`. Even if Submit could be clicked, stale-invalid rows are excluded from what the backend receives, so the new clearing logic can never see them.

Because of (1)+(2), the shipped fix only fully resolves the friction when a blend's rows never got persisted as invalid in Flatfile in the first place, or when the whole sheet's rows are otherwise valid enough for Submit to be clickable. It does not resolve the reported case end-to-end on its own.

## Recommended full fix (not implemented)

Fable's recommendation, as one combined change (each piece is unsafe alone):
1. Drop `hasAllValid` from the Submit action (keep `hasData`) — the backend already re-validates everything at submit time.
2. Change the fetch in `processFilesInBackgrond` to not filter to `valid`-only for the `BulkUploadMatLibV2` sheet slug specifically (other 7 worksheets keep current behavior) — otherwise dropping `hasAllValid` reopens the exact orphaned-constituent hazard the atomic-blend design exists to prevent (a partial blend would reach the processor).
3. The shipped `clearStaleValidationMessages` generalization (already done) is a prerequisite for both — without it, steps 1–2 would surface the accumulation bug even more visibly.
4. After clearing this logic's own messages, if a record still carries an error from another source (e.g. Flatfile's native `required`/Blueprint constraints), treat that as a blend failure too — needed because native-constraint violations would otherwise slip through the `hasAllValid` removal.

Adjacent, separately-noted hazard (not caused by this): blends are grouped per fetched page (`pageSize: 100`), so a blend whose rows straddle a page boundary is already processable as two partial blends today. Not worsened by the above, but worth knowing if hardening blend atomicity further.

## Related

See [[higg-44104-blend-error-pointer]] if that memory exists — this follow-up sits on top of the original blend-error-pointer fix from the same ticket.
