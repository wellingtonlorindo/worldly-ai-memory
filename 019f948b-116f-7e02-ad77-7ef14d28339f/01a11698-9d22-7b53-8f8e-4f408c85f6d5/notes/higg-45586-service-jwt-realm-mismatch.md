---
tags:
- HIGG-45586
- HIGG-45353
- keycloak
- service-jwt
- qa
- resolved
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.6.1
  at: 2026-10-09T14:55:46Z
---
# HIGG-45586: PL service token and the Keycloak realm (RESOLVED 2026-10-09)

The service-token blocker for manual QA of `POST /account-service/v1/internal/claim-reviews` (HIGG-45353, HIGG-44785) is resolved for the **development** environment only.

## Timeline
- 2026-10-07: DevOps (Greys Santander) created client `worldly-projection-layer` in the **`development-internal`** realm. Its token had the right claims (`aud: worldly-api-callback`, `scope: account-service-internal`, `azp: worldly-projection-layer`) but `iss` was `.../realms/development-internal`. The API answered `401 InvalidToken`, with or without a `Bearer ` prefix.
- 2026-10-09: the client was re-created in the **`development`** realm. Its token (`iss` = `.../realms/development`) passes the guard. An unknown-org probe returned `404 organization_not_found`, and the internal-route scenarios then ran (see the QA findings page).

## Why the first token failed
- `serviceJwtAuthentication` calls `verifyKeycloakToken` in `ts/middleware/validate-jwt.ts`. It verifies against ONE pinned cert, `config.jwtCertKeycloak`, then requires `iss === config.keycloak.issuer`. No JWKS fetch, no `kid` lookup.
- The realms sign with different keys (`development` kid `PSuNY6...`, `development-internal` kid `G0zz8W...`), so the signature check failed before the issuer check.
- A wrong audience or scope returns 403. A 401 only says the failure is in `verifyKeycloakToken` and cannot separate a bad signature from a bad issuer.
- An Opus second opinion agreed with this diagnosis.

## Rule to remember
A service client for this API must be created in the same realm the API trusts (`config.keycloak.issuer`). The guard was not changed, and it should not be widened to trust a second realm without a security decision.

## Still open
- Staging's Entitlements Core machine clients use a `staging-internal` realm (`config/v2.api.staging.higg.org.json:5`, `docs/entitlements/core-clients.md`). If `-internal` is intended policy for machine clients, the same mismatch can come back on staging and production. Confirm the intended realm per environment with DevOps.
- The ticket asked for clients in every environment. Only development is confirmed. Staging, demo, asia and production each have their own realm, issuer and cert, and need their own client.
- One pinned cert means a realm key rotation breaks every `service-jwt` caller and Keycloak user until config is redeployed.
- PL and the Celigo callback share audience `worldly-api-callback`, so scope is the only separator. `grantedScopes` also accepts a role of that name from any client's `resource_access`. Check that Celigo's client never gets `account-service-internal`.
- `drp` and `resilience-stack-cn` have an empty issuer, so `service-jwt` always 401s there. `staging-cn` reuses the `development` issuer and cert.
- A valid token still gets 404 if `EnableAccountService` is off globally. Do not read that 404 as an auth failure.
- The ticket's status-lookup path is wrong: the real path is `/account-service/v1/internal/links/status-lookup`.
- `verify` has no `clockTolerance`.
- The optional second client without `account-service-internal` (to test `403 InsufficientScope`) was not delivered.

## Cheap checks
- A token from the PL client should give 404 or 200 on the internal route, never 401.
- Verify a token offline with `jsonwebtoken` against the configured cert to get the exact error without calling the API.
- Check a user token is still live with `GET /account-service/v1/me/organizations` before a run: an expired one returns 401 everywhere and looks like a real result.
