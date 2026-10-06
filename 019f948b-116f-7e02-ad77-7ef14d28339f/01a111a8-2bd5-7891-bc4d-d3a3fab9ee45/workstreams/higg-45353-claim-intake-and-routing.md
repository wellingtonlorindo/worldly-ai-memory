---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-06T19:40:53Z
---
# HIGG-45353 — Claim intake and routing (E5-T1b)

## Status: implemented, draft PR open, QA blocked on stale token

Draft PR: https://github.com/higgco/api.higg.org/pull/10502 (base `Development`, `ai-first`).

Six commits on `HIGG-45353-claim-intake-and-routing`, three PR-slices, all TDD:
1. **ClaimRouter** (`real/claim-router.ts`) + standing-signal + deferred case minter + `createClaimLink` writer.
2. **`AccountEntity.matching.requester`** written from Keycloak GUID (`sub`), never client-authored.
3. **real `ClaimsService`** (`real/claims.ts`) behind `POST /claims` / `GET /claims/:id`.

Verification: 620 account-service tests pass. `npm run tsc` clean except pre-existing `node_modules/survey-data`.

## QA surface map (verified live)

- **Customer `POST /account-service/v1/claims`** (`@Security("jwt")`), body `ICreateClaimRequest`:
  `{"organizationId": "<uuid>", "platform": "wcp|bhive|scm|ffc", "invitationRef?": {...}}`.
  Routes 2 (`proposal_sent`/`applied`) and 3 (`case_minted`/`under_review`).
- **Internal `POST /account-service/v1/internal/claim-reviews`** (`@Security("service-jwt", ["account-service-internal"])`).
  This is the ONLY surface where routing runs; PL doesn't call it yet.
- `claims` seam binds `mock` by default; flip via `HIGG_ACCOUNT_SERVICE_BINDING_CLAIMS=real`
  (already in `.vscode/launch.json` env block, uncommitted, latches at boot).

## QA blocker (2026-10-06): stale JWT — token kid `QU3N...` rotated out

- Token `iss` = keycloak `development` realm, `kid=QU3N...`. Live JWKS only serves `kid=PSuN...`
  (`https://keycloak-auth.non-prod.worldly.io/realms/development/protocol/openid-connect/certs`).
- Verified with openssl: token signature FAILS against `config` `jwtCertKeycloak`.
- Confirmed `jwtCertKeycloak` in `config/v2.api.development.higg.org.json` == current live `PSuN...` key
  (modulus match). So config is correct; the TOKEN is stale (Keycloak signing key rotated).
- `401 InvalidToken` reproduced on BOTH customer `POST /claims` AND `corporate-report` — token rejected everywhere,
  not a code/binding bug.

## Next step
Get a FRESH token from Keycloak `development` realm (`worldly-test-client`). Easiest: grab live
`Authorization` header from the running Angular app (`localhost:4200`) network tab (will carry `kid=PSuN...`).
Then: (1) probe which `claims` binding the restarted server latched (mock → `404 Unknown organization id`
for any org; real → 401/403/409/400 by body), (2) run route-2/route-3 table against live orgs
(`e94603dc-d249-4f03-97c5-a68eff06078c` "Test UK" join; `de5a5e61-a337-4fb5-9541-9f082df5a2ce`
"1test 555550" claimable/source-only).

## Open decisions (confirm with E5 owner)
1. Invited claim + CS decline → route 4 (assumed yes).
2. Standing signal scope = per-claimant CS declines only.
3. PR 2 `requester` needs Keycloak sign-in at account creation, else uninvited PL claims get
   `400 requester_required`.

## Uncommitted local change (do NOT commit)
`.vscode/launch.json` — added `env` block (`HIGG_ACCOUNT_SERVICE_BINDING_ORGANIZATIONS=real`,
`HIGG_ACCOUNT_SERVICE_BINDING_CLAIMS=real`) to "Launch Development API" config.
