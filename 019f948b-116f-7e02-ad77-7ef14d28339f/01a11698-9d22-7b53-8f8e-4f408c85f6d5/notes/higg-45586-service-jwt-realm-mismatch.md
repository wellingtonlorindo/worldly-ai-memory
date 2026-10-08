---
tags:
- HIGG-45586
- HIGG-45353
- keycloak
- service-jwt
- qa
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-08T16:47:43Z
---
# HIGG-45586: PL service token is rejected by the API (realm mismatch)

Status as of 2026-10-08. Blocks manual QA of the internal route `POST /account-service/v1/internal/claim-reviews` (HIGG-45353, HIGG-44785).

## What happened
- DevOps (Greys Santander) created client `worldly-projection-layer` in the **`development-internal`** realm, not `development`. Token URL: `https://keycloak-auth.non-prod.worldly.io/realms/development-internal/protocol/openid-connect/token`. The secret is in a 1Password share restricted to the ticket's named users.
- Its `client_credentials` token has the right claims (`aud: worldly-api-callback`, `scope: account-service-internal`, `azp: worldly-projection-layer`), but `iss` is `.../realms/development-internal`.
- The local API answers `401 InvalidToken`, with or without a `Bearer ` prefix.

## Why
- `serviceJwtAuthentication` calls `verifyKeycloakToken` in `ts/middleware/validate-jwt.ts`. It verifies against ONE pinned cert, `config.jwtCertKeycloak`, then requires `iss === config.keycloak.issuer`. There is no JWKS fetch and no `kid` lookup.
- The realms sign with different keys: `development` uses kid `PSuNY6...`, `development-internal` uses kid `G0zz8W...`. The signature check fails before the issuer check.
- Wrong audience or scope would be 403. A 401 only says the failure is in `verifyKeycloakToken`, so it cannot separate a bad signature from a bad issuer.
- A second opinion (Opus) agreed with this diagnosis.

## Decision still open (do not change the guard unprompted)
- Option A: DevOps creates the client in the `development` realm, like `worldly-api-client`. No code change here.
- Option B: the API trusts both realms. This needs a second cert and issuer, per-env config and tests, and widens the trust boundary, so it is a security decision.
- Counter-signal: staging's Entitlements Core machine clients already use a `staging-internal` realm (`config/v2.api.staging.higg.org.json:5`, `docs/entitlements/core-clients.md`). `-internal` may be the intended policy for machine clients. Ask DevOps whether it is before asking them to move the client.

## Other risks found
- One pinned cert means no rotation: a realm key rotation breaks every `service-jwt` caller until config is redeployed.
- PL and the Celigo callback share audience `worldly-api-callback`, so scope is the only separator. `grantedScopes` also accepts a role of that name from any client's `resource_access`.
- `drp` and `resilience-stack-cn` have an empty issuer, so `service-jwt` always 401s there. `staging-cn` reuses the `development` issuer and cert. Staging, demo, asia and production each need their own client.
- A valid token still gets 404 if `EnableAccountService` is off globally. Do not read that 404 as an auth failure.
- The ticket's status-lookup path is wrong: the real path is `/account-service/v1/internal/links/status-lookup`.
- `verify` has no `clockTolerance`.

## Cheap checks
- Positive test: a `client_credentials` token from `worldly-api-client` (realm `development`) should give 404 or 200 on the internal route, never 401.
- Verify a token offline with `jsonwebtoken` against the configured cert to get the exact error message without calling the API.
