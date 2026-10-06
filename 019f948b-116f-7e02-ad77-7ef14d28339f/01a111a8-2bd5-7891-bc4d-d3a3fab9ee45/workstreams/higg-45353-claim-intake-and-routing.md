---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-06T19:21:54Z
---
# HIGG-45353 — Claim intake and routing (E5-T1b)

## Status: implemented, draft PR open, QA unblocked-manually only

Draft PR: https://github.com/higgco/api.higg.org/pull/10502 (base `Development`, `ai-first` label).

Six commits on branch `HIGG-45353-claim-intake-and-routing`, three PR-slices, all TDD (red→green→refactor, `source env/test && npx jest`):

1. **PR 1 — ClaimRouter** (`ts/managers/account-service/real/claim-router.ts`): full step 0–6 map. Routes: 1 `proposal_sent`, 2 `applied`, 3 `case_minted`, 4 `standing_signal_case_minted`. Standing-signal check runs BEFORE propose/apply (invited too). Removed both 501s; added `requester`/`requester_required`.
   - `AsLinkStandingSignalLookup` (`real/as-link-standing-signal-lookup.ts`): per-claimant CS-declined scan, `STANDING_SIGNAL_SCAN_LIMIT=200`, fail-safe to `decline` on full scan.
   - `DeferredCaseMinter` (`real/deferred-link-ports.ts`): `caseId = "case-<linkId>"`.
   - `createClaimLink` writer on `AsLinkMongo` (side-a acceptance `how: registration`, `originatingSide: "a"`).
2. **PR 2 — `AccountEntity.matching.requester`**: schema + regenerate; account create/update write `requester` from `req._higg.jwt.sub` when `idpSource === "keycloak"`, never client-authored. Threaded as optional `actorGuid` param through `createAccount`/`updateAccount`/`composeAccountEntity` (account.ts) + `account-router.ts`.
3. **PR 3 — real `ClaimsService`** (`real/claims.ts`): `IClaimsService`; `createClaim` (requires `actorGuid` 401, placeholder accountId `<platform>-<uuid>`, calls router, projects side-a-only); `getClaim` (requester or link-capable member). `service-registry.ts` wires `realClaimsFactory()`. `POST /claims` decorators 400/401/403/404/409.

**Verification:** 620 account-service tests pass (42 suites). `npm run tsc` clean except pre-existing `node_modules/survey-data` error (unrelated). `ts/test/account/` has 3 pre-existing Mongo-connect failures (account-invite etc.) — env/network, not this work.

## Open decisions (spec "Still open" — confirm with E5 owner before rollout)
1. Invited claim + CS decline → route 4 (assumed yes here).
2. Standing signal scope = per-claimant, CS declines only.
5. PR 2 `requester` needs Keycloak sign-in at account creation; else uninvited PL claims get 400 `requester_required`.

## QA manual testing — hard blockers (verified live against local :5000)
- **Internal route `POST /account-service/v1/internal/claim-reviews` needs a `service-jwt`** (`aud: worldly-api-callback`, scope `account-service-internal`), NOT a user JWT. User JWT → `403 InvalidAudience` (confirmed). This is the ONLY surface where the new routing runs.
- **`claims` binds `mock` by default** — customer `POST /claims` on a real org returned the mock's `Unknown organization id` 404 (confirmed). So customer surface doesn't exercise PR 3 until binding flipped.
- **PL doesn't call `claim-reviews`** (companion `HIGG-45353-pl-claim-review-caller-spec.md` part A unbuilt).

## What IS reachable + already verified locally (user superuser JWT)
- `EnableAccountService` ON, `organizations` seam real, search returned live orgs (both `join` e.g. `e94603dc-d249-4f03-97c5-a68eff06078c` "Test UK", and `claimable` e.g. `de5a5e61-a337-4fb5-9541-9f082df5a2ce` "1test 555550" — source-only).

## Spec edits (gitignored — local planning artifacts, do NOT commit)
- Appended "QA Manual Testing Guide" to `docs/specs/HIGG-45353-claim-intake-and-routing-spec.md` (modeled on HIGG-44769 + HIGG-44785). Full curl-per-row table; route 4 marked not-manual (seed optional); `matching.requester` section; teardown.

## Uncommitted local change
- `.vscode/launch.json` — added `"env"` block to "Launch Development API" config: `HIGG_ACCOUNT_SERVICE_BINDING_ORGANIZATIONS=real`, `HIGG_ACCOUNT_SERVICE_BINDING_CLAIMS=real`. **Git-tracked, must NOT be staged/committed.** Requires restart (bindings latch at boot) and only unlocks the CUSTOMER surface, not the internal route.

## To unblock full manual QA (next session)
1. Get a `service-jwt` (ask PL Keycloak client owner — same dep as `status-lookup` route).
2. Restart local API from VS Code debug tab with the new `env` block (or `HIGG_ACCOUNT_SERVICE_BINDING_CLAIMS=real source env/dev ts-node ts/index.ts`).
3. Then run: internal 4xx/route table with service-jwt; customer `POST /claims` routes 2/3 with user JWT. Route 4 stays unit-suite only.
