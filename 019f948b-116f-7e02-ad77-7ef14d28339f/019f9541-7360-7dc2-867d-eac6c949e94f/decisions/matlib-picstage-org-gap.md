---
tags:
- matlib
- pic
- debug
- root-cause
tier: semantic
---
# PicCalculateStageEntry org field missing — root cause

`PicCalculateStageEntry.organization` (returned by `GET /corporate-report/{accountID}/matlib/materials/{materialID}`) is populated by exactly one code path: `MatlibManager.toPicCalculateStages` (`ts/lib/matlib/matlib-manager.ts:1028-1055`), called only from the UI create/calculate flow (`POST /matlib/materials`), and only when the FE has already supplied `stage.organizationId` (via manufacturer "match-universe-account" selection) — resolved through `findOrganizationById`.

Every other stage-construction path never sets `organization`:
- `MatlibManager.resolvedActivityToStageEntry` (`ts/lib/matlib/matlib-manager.ts:665-678`) — used by `setDefaultLifecycle` (the default path run whenever a base material is first added) and directly by the flatfile MatLib V2 bulk-upload mapper (`flatfile-matlib-v2-record-mapper.ts` `buildSelectedStageEntry`).

**Why:** the upstream `msi-in-the-universe` GraphQL schema types feeding these paths (`MatLibResolvedActivity`, `MatLibStageActivity`) carry no organization/account identifier at all — only a free-text `manufacturer` name string. There's nothing to plumb through without upstream schema changes.

The GET endpoint (`ts/routes/corporate-report-router.ts` `getMatlibMaterial` → `matlibManager.getMaterialById` → `DataServices.picMaterialLibraryCouch.getEntityAccount`) is a raw passthrough of the persisted CouchDB doc — no enrichment at read time — so it faithfully reflects whatever was (or wasn't) set at write time.

**Net effect:** since most materials get their initial stages via default-lifecycle assignment or flatfile import (not the narrow explicit-organizationId calculate flow), `organization` is null/absent for the large majority of materials. This is a cross-repo data-availability gap (msi-in-the-universe schema + FE), not a fixable bug in api.higg.org alone.

**Options on the table (undecided as of 2026-08-18):**
1. Accept as by-design — org only available when user explicitly goes through manufacturer matching.
2. Escalate cross-repo — ticket msi-in-the-universe to expose org identity on activity nodes, FE to always resolve+send organizationId for manufacturer-sourced stages.
3. Narrow further with a real accountID/materialID to confirm which flow produced a specific document before deciding.

Full investigation trail: `.planning/debug/matlib-material-org-missing.md` (gsd-debug session).
