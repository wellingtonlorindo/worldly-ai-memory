---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-06T18:36:21Z
---
# Plan: seed script for Account Service real-seam data (FE unblock)

Status: PLANNING (Opus review pending). Owner: Wellington. Date: 2026-10-06.

## Problem
Account Service seams bind `mock` (JSON fixtures) or `real` (PL + Mongo). FE dev who sets
bindings to `real` sees empty/404 unless PL + Mongo are seeded with fixture-equivalent data.
Goal: seed so the FE works "just like it does today with mocks".

## The two stores real seams read
1. **Projection Layer (PL)** — `worldly-projection-layer` repo. GraphQL read-only over Neo4j
   (`OrganizationNode`, `WorldlyAccountNode` + `BaseAccountNode` co-label, `NetworkMemberNode`) +
   Postgres (`organization`). api.higg.org reads via `ProjectionLayerClient`.
2. **MongoDB** — `as_links` via `AsLinkMongo`.

## Decisive findings
- Real org profiles are **composed** from a real user's accounts
  (`OrganizationProfileComposer.compose(email)` → `UserAccountSources.enumerateAccounts` →
  `Pl.lookupOrganizations` → join link status). No flat org store.
- **Identity steering dies when all seams go real**: `X-Mock-Persona` resolver is only installed
  while a seam is mock; then identity = real JWT (`jwt.sub`+`jwt.email`). So "FE works like mocks"
  under real bindings requires persona steering still active, OR real accounts owned by the dev.
- PL writable two ways:
  - **Option A project-through** — fake account entities → `WorldlyBackfill` → `worldly-sync` queue →
    **worker process** projects (mints `WorldlyAccountNode` + `OrganizationNode`). Canonical, needs a
    running worker.
  - **Option B hand-write** — call `OrganizationNeo4jStore.save()` + `AccountNeo4jStore.save()`
    directly (MUST write `BaseAccountNode` co-label + respect label-scoped indexes).

## Current real-seam coverage (from PRs)
- `organizations` real: EXISTS (OrganizationsService + composer).
- `links` real: EXISTS (LinksService). `getLinkCandidates`/`createLink`/`withdrawLink` = 501 Group B.
- `claims` real: LANDING via PR #10502 (E5-T1b, current ticket) — `ClaimsService` behind
  `POST /claims` + `GET /claims/:id`, `bindings.claims: "real"`. PR #10485 added the invited-claim
  internal route (`POST /internal/claim-reviews`, `ClaimRouter`).
- `platformAccounts` real: NOT EXISTS (mock only).

## Open decisions / blockers
1. **PL fork** — user said "investigate PL projection first"; investigation done (see note
   [[account-service-mock-vs-real-seed]] + this page). Recommendation leans Option B hand-write for
   a self-contained local seed, unless PL worker is acceptable to run.
2. **Identity** — user first said "fake-but-consistent ids", then clarified "as long as FE works
   like today with mocks". These conflict: fake ids → real FE login sees nothing. Resolve by either
   (a) keep `X-Mock-Persona` steering alive in a dev-only real mode, or (b) map to real dev accounts.

## Plan shape (two parts)
1. `api.higg.org`: new `ts/bin/seed-account-service.ts` translating `AccountServiceFixtures` →
   `as_links` via `DataServices.asLinkMongo` (pattern: `ts/bin/seed-as-links.ts`, bandContext runId
   tag, `--dryrun`/`--cleanup`).
2. `worldly-projection-layer`: seed/backfill materializing fixture orgs in Neo4j (+ `organization`
   Postgres table if required), respecting co-label + label-scoped index invariants.

## Verification
- `source env/test && npx jest ts/test/account-service/real/*`
- Curl real-bound routes with a real/steered identity; confirm fixture-equivalent responses.

Related: [[account-service-tickets-overview]], [[account-service-mock-vs-real-seed]], [[account-service-shopping-boundary]].
