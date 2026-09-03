---
tags:
- higg-42667
- msi
- chemistry-score
- score-comparison
- resolved
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-07-30T16:56:54Z
---
# HIGG-42667: MSI import chemistry-score divergence — RESOLVED

## Resolution
The chemistry divergence found earlier (see history below) is fixed. Root cause: the source
export's **finished-material chemistry certifications** (bluesign, GOTS, GRS, Oeko-Tex,
Cradle to Cradle — what a brand certifies a material against) were never read at all. They
don't appear on the Custom Material Constituents sheet (which the script reads); they're
written once per material as a deduped union on the **Aggregated Custom Materials** sheet's
"Full Material Chemistry Certifications" column (`ts/lib/msi-excel.ts`
`getAllFinishedMaterialChemCerts`, ~line 267-271, 718-728). `chemCertFactor` in
`node_modules/msi-utils/js/manager.js` `calculateProcessScores` (~line 566-572) checks
finished-material certs **first**; with none read, every material lost whatever discount the
source had, inflating chemistry ~2x for certified materials.

**Fix shipped** (`ts/bin/import-anonymized-msi-materials.ts`): added
`extractFinishedMaterialChemCertsByMaterialId()` (reads the Aggregated sheet, optional —
missing sheet is not a hard failure), reverse-mapping via `parseFinishedMaterialChemCerts()` /
`parseProcessChemCerts()` (title string → enum, built from `MsiUtils.getChemCertTitle()`
itself rather than hardcoded strings), applied onto `BaseMaterialEntity.finishedMaterialChemCertsSelected`
and threaded through `buildBaseMaterialFromRow → buildMaterialEntity → buildAllMaterialEntities
→ runImport`. Also added `collectProcessDetails()`/`applyProcessDetails()` to carry
per-process `Process N Chemistry Certifications` (raw-material/facility certs) and
`Process N Transportation distance/unit/Mode` onto the rebuilt `MaterialSelectedProcess` —
previously always silently defaulted (200km/truck, no cert). None of this is
customer-identifying: cert titles and transport unit/mode are closed, standardized
vocabularies (~11 cert values, 2 transport units, 4 modes).

## Before/after (same real REI/Patagonia-shaped export, 1399 constituent rows, 827 materials)
| | Before | After |
|---|---|---|
| chemistry/chemistryPts warn rate (\|Δ%\|>5) | 64.9% (908/1399) | 7.3% (102/1399) |
| chemistry avg \|Δ%\| | 49.3% | 2.15% |
| chemistry max \|Δ%\| | 118.6% | 49.9% |
| all other 5 categories | already ≤0.2% avg | unchanged, still ≤0.2% avg |

Zero "unrecognized certification title" warnings on this export — every cert title present
matched the reverse-lookup vocabulary. The residual ~7.3%/2.15% avg is a much smaller,
plausibly genuine catalog-drift-style gap (unconfirmed, not worth chasing further given the
scale of improvement already achieved).

## Retracted finding (kept for history, do not re-propose as originally stated)
Original hypothesis "missing chemistry certs cause the divergence" was tested against one
concrete material (`5fb2d1ecad880c000aa5a0b6`, Polyester fabric) by checking only the
Constituents-sheet `Process N Chemistry Certifications` column — found blank on both source
and recompute, seemingly ruling out certs entirely. That check missed the *other* cert
channel (Aggregated sheet, finished-material certs) — that same material's Aggregated-sheet
row has `"Full Material Chemistry Certifications": "Bluesign Certified [Material]"`. Lesson:
this export has **two independent, non-overlapping chemistry-cert columns on two different
sheets** — checking one and finding it blank does not rule out the other.

## Also changed in the same pass
- Added absolute `delta` (computed − source) alongside `deltaPct` in the score-comparison log
  (`buildScoreComparison`/`logScoreComparison`) — deltaPct alone amplifies noise on small
  denominators like chemistry.
- `MsiManager.createMaterial()` (`ts/managers/msi/index.ts:743`) was reconsidered again (per
  PR #10007 reviewer's original suggestion) and rejected again, with two objections found to
  be *worse* than originally documented: `isMaterialNameAvailable` breaks re-runnability (the
  script's synthesized names restart at index 1 every run — a second run into the same account
  would throw `BAD_REQUEST` per material), and `syncMaterial()` fires even under `dryRun=true`
  (not just on real create). Direct-build architecture confirmed as the right call, unchanged.

## Spec
`docs/specs/HIGG-42667-anonymized-msi-material-import-spec.md` — this fix is a follow-on to the
already-implemented, spec-compliant script; the spec's "explicit-identifiers-only, not strict
allowlist" framing was retroactively validated by this investigation. Consider updating the
spec's Data Model / Security Considerations sections to reflect that chemistry certs and
transport fields are now imported (not dropped), if/when the spec doc itself gets revised.
