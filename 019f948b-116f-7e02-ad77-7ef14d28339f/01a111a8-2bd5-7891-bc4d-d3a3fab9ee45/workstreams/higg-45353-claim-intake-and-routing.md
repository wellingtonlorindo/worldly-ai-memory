---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-06T19:48:51Z
---
# HIGG-45353 — Claim intake and routing (E5-T1b)

## Status: implemented, draft PR #10502 open, customer-surface QA run live 2026-10-06

Draft PR: https://github.com/higgco/api.higg.org/pull/10502 (base `Development`, `ai-first`).

Three slices on branch `HIGG-45353-claim-intake-and-routing`: ClaimRouter, `matching.requester`, real `ClaimsService`. 620 account-service tests pass.

## CORRECTION: earlier "stale token" diagnosis was WRONG
The user's token was valid. Local API (`:5000`) auth for `@Security("jwt")` takes the raw token in `Authorization` with NO `Bearer` prefix, and I had (a) sent `Bearer`, and (b) hand-retyped the JWT and corrupted its signature. Always write the pasted token to a file once and read it from there; send `Authorization: $(cat token)`.

## Customer POST /account-service/v1/claims — live results (superuser Keycloak JWT, `claims` bound real)
- Unknown org -> 404 `organization_not_found` (real shape, not mock).
- Org `e94603dc...` "Test UK", platform bhive -> 200 `pending` / `proposal_sent`; GET /claims/:id returns same.
- Org `de5a5e61...` "1test 555550", platform wcp -> 409 `organization_holds_account`, BUT org search for wcp lists it as `claimable`. Search path uses the search-hit `worldlyAccountId`; claim path uses `getOrganization` (GraphQL) `worldlyAccountId`. Possible search/claim eligibility inconsistency — NOT yet root-caused.
- `de5a5e61` as bhive / ffc -> 200 `proposal_sent`.
- Blank `invitationRef.accountId` -> 400; no auth -> 401; bad platform -> 400 validation; unknown claim id -> 404.
- Repeating the same claim creates a NEW link each time (fresh placeholder accountId by design).

## NOT yet exercised
- Route 3 `case_minted` / `under_review` never reached (every success returned `proposal_sent`).
- Route 2 `applied` (invited claim), route 4 (standing signal), `matching.requester` write, 403 member-below-link-capable.
- Internal `POST /internal/claim-reviews` still needs a `service-jwt`.

## Test data left in shared dev Mongo (links, pending): 261a5140-c2f6-4bec-b08e-11b40b64ce15, b355101d-ff22-43fa-9326-17678dcd7b41 (Test UK/bhive); 1252db13-5fbe-4c2d-81a0-2c0fd9d27fd5 (de5a5e61/bhive); 0a52c255-467c-41dc-a597-0c06e81a7827 (de5a5e61/ffc).

## Uncommitted local change (do NOT commit)
`.vscode/launch.json` env block (`HIGG_ACCOUNT_SERVICE_BINDING_ORGANIZATIONS=real`, `..._CLAIMS=real`).

## Open decisions (confirm with E5 owner)
1. Invited claim + CS decline -> route 4 (assumed yes).
2. Standing signal scope = per-claimant CS declines only.
3. `requester` needs Keycloak sign-in at account creation, else uninvited PL claims get `400 requester_required`.
