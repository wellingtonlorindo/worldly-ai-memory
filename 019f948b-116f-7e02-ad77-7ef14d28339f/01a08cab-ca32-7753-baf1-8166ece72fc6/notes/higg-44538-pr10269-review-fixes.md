---
tags:
- HIGG-44538
- PR-10269
- account-service
- code-review
- fable-review
pinned: true
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-14T20:52:07Z
---
# HIGG-44538 / PR #10269 — Fable review findings (2026-09-14, not yet fixed)

Context: branch HIGG-44538, PR #10269 (Account Service E0 mock surface). Addressed two
post-merge PR comments from natiels (2026-09-14): the `pendingWith` → `hold` +
`awaitingConfirmationFrom` contract change, and a 7-item review (items 1-4 blocking:
me/organizations full profiles, resolve membership check, claims routing table, X-Mock-Fail
bounds/auth; items 5-6 behavior divergences: drop 409-on-second-claim + derive claimPending
live, decline eligibility == confirm eligibility; item 7 optional: one `confirmed`-state fixture).

All of that was implemented, tests added (109/109 account-service tests green, full repo
`npm run build` + `tsc` clean, full `npx jest` run shows only pre-existing/environmental
failures unrelated to this branch — MongoDB-secret-access crashes in `ts/test/account/*`,
`ts/test/brm/share*`, `ts/test/adoption/*`, plus one unrelated pre-existing timezone bug in
`ts/test/es/elasticsearch-utils.test.ts`).

Then dispatched a Fable subagent (general-purpose, model: fable) to critically review the diff
before committing. **These findings were NOT yet applied — still need triage/fix before commit:**

## Real bugs (should fix)

1. **`GET /organizations/resolve`'s gate re-derivation is a no-op.**
   `ts/routes/account-service-v1-router.ts` (resolveOrganization handler): calls
   `AccountServiceContext.from(req)` (no org) first, which gates *globally*. Per
   `ts/managers/config-keys.ts` `isEnabledForAccountOrGloballySync`: if the key `isMadeGlobal`,
   result is `globalValue` regardless of accountId (so the later org-scoped call is always the
   same answer — no-op); if NOT global, the first un-scoped call needs `accountId` to be an
   `Account`-tokenType key, but org is unknown yet, so global-check with `organizationId=undefined`
   returns false as soon as the key is Account-scoped and not made global — meaning the whole route
   404s before the lookup ever runs, in the exact canary-rollout scenario the fix was meant to
   support. Net effect: the second `AccountServiceContext.from(req, profile.organizationId)` call
   added to satisfy the review comment never actually changes behavior; the per-org canary path is
   unreachable either way. Needs restructuring: resolve identity without gating, do the org lookup,
   *then* gate scoped to the resolved org, only after which return/deny. No test can currently prove
   this correctly (would be vacuous against current code).

2. **`under_review` claims aren't put on `hold`, so the claimant can self-confirm out of CS review.**
   `ts/managers/account-service/mock/claims.ts` createClaim: the (source-records-only, uninvited) →
   `under_review` cell creates a plain `proposed` `claim_uninvited` link with no `hold`. Since
   `sideA === sideB` (both are the claimant's org — no anchor case), `viewerSidesFor` puts the
   claimant on both sides, so `viewerCanConfirm`/`viewerCanConfirm`-derived `viewerCanDecline` are
   both true — the claimant can confirm (or decline) their own under-review claim, bypassing CS
   review entirely. Compare `links.ts` createLink, which DOES set `hold: LinkHold.InternalReview`
   for its analogous CS-review case. Fix: set `hold: LinkHold.InternalReview` on the `under_review`
   branch of createClaim too (only when `!autoApplies && !holdsAccount && !invited`, i.e. the
   CS-review cell).

## Semantics / medium issues

3. **`awaitingConfirmationFrom` "sideA is the accepted originator" rule is inverted for claims.**
   `link-mapping.ts` docstring claims sideA is always the already-accepted originator for
   either-side origins. True for `createLink` (sideA = requester's anchor). For `createClaim`,
   sideA is the org's *existing* anchor Account and sideB is the *newly provisioned* one — the
   claimant (holder of the existing anchor) is the originator, so the outstanding party for a
   `proposal_sent` claim is really the *new* account's future holders — which happens to still be
   sideB in the mock's shape, so the code's actual behavior (list sideB) is arguably still right,
   but the justification/docstring reasoning is wrong for the claims case and should be corrected
   or the two origins' logic should be revisited together with finding 2.

4. **Misleading comment — no "same-platform guard" exists.**
   `claims.ts` comment claims a second pending claim "lands on the same-platform guard once the
   first confirms" (from the natiels review text) — but `confirmLink` has no such check. Two
   pending claims on the same org/platform can both independently reach `applied` with distinct
   provisioned account ids. Either implement a minimal guard or stop asserting one exists in the
   comment.

5. **Fixture inconsistency — `LINK-CONFIRMED-1` (the new item-7 fixture) contradicts `organizations.json`.**
   `account-service-fixtures/links.json` LINK-CONFIRMED-1 has ORG-GAMMA holding `WCP-G1` confirmed
   on wcp, but `organizations.json` still shows ORG-GAMMA's wcp row as `not_linked`. Breaks the
   README's own stated invariant that org platform rows and links.json cross-reference correctly.
   Consequence: `GET /organizations/ORG-GAMMA` shows wcp not_linked while `/links` shows a confirmed
   wcp link; `/organizations/resolve?platform=wcp&accountId=WCP-G1` 404s. Fix: add a `pending` (not
   `linked`) wcp row on ORG-GAMMA in organizations.json referencing `linkId: LINK-CONFIRMED-1` (a
   `pending` row is fine per how `holdsAccount`/`withPlatformEligibility` only check `Linked`, so
   routing-table tests using ORG-GAMMA as the no-anchor org stay correct). Also add a router-level
   test reading the actual fixture link and asserting `viewerCanConfirm: false`,
   `viewerCanDecline: false`, `awaitingConfirmationFrom: []` — only a synthetic unit test currently
   covers the Confirmed state.

## Low priority

6. **`fault-injection-middleware.ts`'s bare `res.sendStatus(401)` diverges from the app's normal
   error envelope** (`errorMiddleware` renders `{status, message, errorFields}` JSON elsewhere).
   Prefer `next(error)` so the shared middleware handles it — the mock's 401 shape should match
   what every other route returns, since FE codegens against this surface.

## Nits (optional)

- `LINK-UNAUTHORIZED-1` fixture is `claim_uninvited` spanning ORG-ALPHA→ORG-BETA, which contradicts
  the claim model's same-org sideA/sideB invariant, and is now load-bearing: `hasPendingClaim`
  makes `claimPending: true` for ORG-BETA too, contradicting `organization-search.json`'s seeded
  `false` for Beta Mills. No test currently asserts either value for Beta.
- `account-service-contracts-d.ts` `resolveOrganization` JSDoc still only documents 404; needs a
  403 line added.
- `links.ts` decline rejection message text ("Link is not in a pending state") is slightly
  misleading when the real reason is a hold or an already-confirmed side; reasonCode is correct
  though.
- `link-mapping.test.ts`'s "regardless of confirmedSides" test title only exercises `{false,false}`
  — could add a `{true,false}` case for the either-side-origin test to actually prove the claim.
- No test locks the auth-before-delay ordering in fault-injection-middleware (a
  rejected-auth + `x-mock-delay-ms` case asserting `setTimeout` was never called).
- flag-on (`EnableAccountServiceAutoApproval`) + `holdsAccount=true` combination in the claims
  routing table is implied correct by the code but has no explicit test.
- `LinkHold` docstring in `account-service-d.ts` narrates the removed `LinkPendingWith` — house
  style (`.claude/rules/code-quality.md`) says remove history, not annotate it. Harmless, could trim.

## Verification already done by Fable (still valid)
- `tsc --noEmit` clean, eslint clean on all changed files, 11 suites / 109 tests green
  (`ts/test/account-service`).
- No stale `pendingWith`/`LinkPendingWith` references remain outside prose/docstrings.
- Generated files (`ts/routes.ts`, both swagger.json files) look like plausible regenerations:
  `POST /claims` responses = `['200']` only, `GET /organizations/resolve` = `['200','403']`,
  `IMyOrganizationsResponse.organizations` → `IOrganizationProfile[]`, `ILinkSummary.required`
  includes `awaitingConfirmationFrom`.

## Next steps (not yet done as of this note)
Fix items 1, 2, 5 at minimum (real bugs / fixture inconsistency) before committing. Consider 3, 4,
6 and the nits based on time. Then commit via the `commit-messages` skill, including the
regenerated swagger files (`dist/swagger.json`, `dist-account-service-api/swagger.json`,
`ts/routes.ts`), per explicit user instruction to commit the swagger files.
