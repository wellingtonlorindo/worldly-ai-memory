---
tags:
- account-service
- HIGG-44538
- testing
- jwt
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-11T19:03:28Z
---
# Account Service (HIGG-44538) — minting test JWTs for fixture users

`account-service-fixtures/users.json` seeds 4 fixture users the mock `AccountServices` layer
resolves identity against by JWT `sub`:

| sub | memberOf |
| --- | --- |
| `auth0\|acct-svc-ana` | `ORG-ALPHA` |
| `auth0\|acct-svc-ben` | `ORG-BETA` |
| `auth0\|acct-svc-cam` | `ORG-ALPHA`, `ORG-BETA` |
| `auth0\|acct-svc-dee` | `ORG-GAMMA` |

These are **not real Auth0 accounts** — no need to create them in Auth0. `jwt` security
(`ts/middleware/validate-jwt.ts`) verifies a real RS256 signature, so a hand-crafted token with an
arbitrary `sub` only verifies if it's signed with a key matching whatever cert the running server's
`jwtCert` config points to.

**The trick:** `config/v2.api.test.higg.org.json`'s `jwtCert` is exactly the self-signed cert in
`ts/test/fixtures/test-crt/certificate.pem` — and the matching private key
(`ts/test/fixtures/test-crt/key.pem`) is checked into the repo. `Provision2.createTestJwt()`
(`ts/test/utils/provision2.ts:140-172`) already signs arbitrary-`sub` JWTs with that key for tests.

So: **a server started with `source env/test` (also port 5000, same as env/dev) will accept a
locally-minted JWT with `sub: "auth0|acct-svc-ana"` etc.** A server started with `source env/dev`
uses the real `apparelcoalition.auth0.com` Auth0 dev-tenant cert instead — locally-minted tokens
are rejected there, and only a genuine Auth0-issued token for a real user works.

The account-service mock endpoints don't touch Couch/ES/PG at all (pure in-memory fixtures), so
switching the local server to `env/test` for this feature carries no real data-access risk — it's
just picking which JWT cert is trusted.

**To mint tokens:** a small script that imports `Provision2` from `ts/test/utils/provision2` and
calls `Provision2.createTestJwt({ sub, email })` per fixture user, run via `ts-node` from inside the
project (an external/scratchpad path fails module resolution — must live under `ts/`).

Related: [[qa-guide-account-service-e0]] if that page exists, and the QA Manual Testing Guide
section of `docs/briefs/HIGG-44537-account-service-e0-fe-facing-mock-and-route-contract-brief.md`
(gitignored, main checkout only).
