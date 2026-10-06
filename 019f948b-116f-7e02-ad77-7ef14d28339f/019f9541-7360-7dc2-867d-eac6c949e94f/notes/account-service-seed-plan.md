---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-06T18:41:48Z
---
# Plan: seed script for Account Service real-seam data (FE unblock)

Status: PLANNED — Opus-reviewed. Owner: Wellington. Date: 2026-10-06.

## Problem
Account Service seams bind `mock` (JSON fixtures) or `real` (external stores). FE dev who sets
bindings to `real` sees empty unless the full store chain is seeded. Goal: seed so FE works
"just like it does today with mocks".

## Store chain (CORRECTED by Opus review — the old "PL + as_links" model was wrong)

A real read resolves through these, in order (empty at any hop stops the chain):

| Read | Store |
|---|---|
| held accounts (`WcpUserAccountSource.enumerateAccounts`) | api.higg.org Mongo `user`+`account`, ES `accountES.getUserAccounts`, `accountCouch.getEntityIfExists` |
| org id per account (`POST /organizations/lookup`) | **PL's own Mongo** `organizations` (`PROJECTION_MONGO_DATABASE`) — NOT Neo4j |
| org detail / aliasIds (GraphQL `organizationNodes`) | PL Neo4j `OrganizationNode` |
| search/summary (REST `/organizations/search`) | PL Neo4j vector index `base_account_embedding` (text fallback) |
| link list/detail/confirm/decline | api.higg.org Mongo `as_links` |

Consequences:
1. Composer returns `[]` unless api.higg.org Mongo+ES hold `AccountEntity` with dev's
   `UserEntity._id` in `users` (Accepted owner/admin). PL + as_links alone = nothing.
2. `lookupOrganizations` reads PL Mongo `organizations`, NOT Neo4j — hand-writing Neo4j (Option B)
   breaks lookup → breaks getMyOrganizations/getOrganization/resolveOrgId/all Links gates.
3. Identity = JWT `email`, not X-Mock-Persona/persona once real. "fake ids" must mean fake
   accounts/orgs/links + dev's REAL `UserEntity._id`. Fake users can't log in via Auth0.
4. Real coverage: organizations+links real (getLinkCandidates/createLink/withdraw = 501).
   claims real = landing via PR #10502 (E5-T1b). platformAccounts = no real impl, stays mock.

## Decision: Option A (project-through), driven from api.higg.org
Write api.higg.org AccountEntity via `accountCouch.createEntity` → fires `projectChange` →
`ProjectionLayerQueue.project` → PL `AccountHandler` mints everything (Neo4j AccountNode both
labels, PL Mongo organizations, OrganizationNode+HAS_WORLDLY_ACCOUNT, Postgres organization,
embedding). Never touch PL internals.

Costs to handle:
- Org ids = random UUIDs (non-deterministic). Poll `lookupOrganizations` by account id (like
  `seed-as-links.ts:resolveOrganizationId`) → write as_links keyed on resolved org ids.
- PL worker must run + `HIGG_PROJECTION_QUEUE_ENABLED`+Redis; seed polls + timeout fail-loud.
- Cleanup partial: AccountHandler soft-deletes (status flag); PL orgs persist. `--cleanup` =
  deactivate+tagn+remove links.
- Embeddings: deployed dev PL, nodes w/o `embedding` silently absent from vector search. Local
  `EMBEDDINGS_DISABLED=true` → text fallback (LIMIT 1000, unordered).

## Plan shape — `ts/bin/seed-account-service.ts` (single repo)
1. `--email` → resolve real `UserEntity._id`.
2. Create tagged AccountEntity docs (fake ids, name-prefix tag) via accountCouch, user in `users`
   Accepted owner/admin (non-admin for `vic` case); `accountES.indexAccount({refresh})`.
3. Wait PL projection (poll lookupOrganizations, timeout fail-loud).
4. Seed `as_links` vs resolved org ids (reuse seed-as-links: state layering + bandContext runId).
5. Claimable/orphan search results: SMT network-member entities to mint orphan orgs, or defer.
6. `--cleanup` = deactivate+tagn+remove links (PL orgs persist).

## Preconditions (fail-fast)
EnableAccountService global; HIGG_PROJECTION_GRAPHQL_ENABLED+url+key; HIGG_ACCOUNT_SERVICE_MONGODB_NAME;
HIGG_ACCOUNT_SERVICE_BINDING_ORGANIZATIONS/_LINKS=real, claims/platformAccounts=mock.

## Mock-parity deviations (tell FE)
Real org = UUID + WCP displayName/address/country; fixture orgId/orgType/city have no real source.
Mock claims write in-memory LinkStore, not as_links.

Related: [[account-service-tickets-overview]], [[account-service-mock-vs-real-seed]], [[account-service-shopping-boundary]].
