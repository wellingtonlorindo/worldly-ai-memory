---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-06T18:35:03Z
---
# Account Service: mock vs real seams and the seed gap

Durable notes on how the Account Service's `mock`/`real` binding split works, and what a
seed script must populate to unblock FE dev on `real` bindings.

## Bindings
Per-seam, selected by `server_config.accountService.bindings.*`, env-overridable via
`HIGG_ACCOUNT_SERVICE_BINDING_{ORGANIZATIONS,LINKS,CLAIMS,PLATFORM_ACCOUNTS}`.
`AccountServices.initialize()` (ts/managers/account-service/service-registry.ts) binds each seam
independently (mock-now/real-later). `EnableAccountService` (Account-scoped) gates the whole
`account-service/v1` surface (404 when off).

## mock seam
- Serves `account-service-fixtures/*.json` (users/orgs/links/claims/organization-search/
  link-candidates/platform-account-search), loaded only via `ts/managers/account-service/fixtures.ts`.
- Identity steering: `X-Mock-Persona` header → fixture `authUserId` (default persona `ana`).
  Installed by `AccountServices.initialize()` via `AccountServiceContext.resolveIdentityWith(...)`
  **only while any seam still binds mock**.

## real seam — reads two external stores
1. **Projection Layer (PL)** — separate repo `worldly-projection-layer`. GraphQL read-only over
   **Neo4j** (`OrganizationNode`, `WorldlyAccountNode` with `BaseAccountNode` co-label,
   `NetworkMemberNode`) + **Postgres** (`organization` table). api.higg.org reads via
   `ProjectionLayerClient` (ts/lib/projection-layer/*).
2. **MongoDB** — `as_links` collection via `AsLinkMongo` (ts/mongo/as-link-mongo.ts).

### Key: real org profiles are COMPOSED, not stored flat
`OrganizationProfileComposer.compose(email)` (real/organization-profile-composer.ts):
enumerateAccounts(email) via `UserAccountSources.registeredSources()` (WCP + BHive) →
`Pl.lookupOrganizations({platform,accountId})` → group by org → join pending-link status from
`AsLinkMongo`. So an org profile exists only if a real user's account resolves to it in PL.

### Identity dies when all seams go real
`AccountServiceContext.identityOnly(req)` falls back to the real JWT (`jwt.sub` + `jwt.email`)
once the identityResolver is uninstalled. `actorGuid` is only set when `idpSource === "keycloak"`.
`X-Mock-Persona` steering disappears when every seam flips real — a real FE login must own the
accounts the seed writes, or the FE sees nothing.

## PL is writable, but orgs are normally projected
- `AccountNeo4jStore.save()` writes `WorldlyAccountNode` + `BaseAccountNode` co-label
  (MUST write both labels or node invisible to GraphQL; via `buildMergeQuery(..., ["BaseAccountNode"])`).
- `OrganizationNeo4jStore.save()` MERGEs `OrganizationNode` + wires
  `(:OrganizationNode)-[:HAS_WORLDLY_ACCOUNT]->(:WorldlyAccountNode)`.
- Canonical org minting happens in a projection handler fed by `WorldlyBackfill`
  (src/backfills/sources/worldly-backfill.ts) — fetches account ids from Worldly source Mongo,
  enqueues to `worldly-sync` queue, a **worker** (`npm run start`, `RUN_MODE=worker`) projects.
- Seed-script precedent: `npm run seed:axion-organization-outputs`; backfill-list entries 29/30
  populate the `organization` Postgres table.
- PL CLAUDE.md gotchas: multi-label nodes (co-label), label-scoped indexes, backfill-list ids unique.

## Existing link seed
`ts/bin/seed-as-links.ts` seeds `as_links` directly (`DataServices.asLinkMongo.createLink` +
`replaceLink`), tags `bandContext: {band, runId}` for `--cleanup`, has `--dryrun`. Links half only —
no PL orgs.

## The seed gap (what the seed script must solve)
Populate BOTH PL orgs (so lookup/search/getOrganization resolve) AND `as_links` (so links resolve),
mapped so a real keyed identity sees fixture-equivalent data. Open decision: project-through
(Option A, canonical, needs worker) vs hand-write PL graph (Option B, store-level, respects
co-label + label-scoped index invariants). See decision in the plan file
`glowing-exploring-seal.md`.

Related: [[account-service-tickets-overview]], [[account-service-shopping-boundary]].
