---
kind: plan
entities:
- seed-account-service
- higg-43344
- pr 10505
tags:
- higg-43344
- seed
- plan
- account-service
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.6.1
  at: 2026-10-15T18:18:43Z
---
# HIGG-43344 seed script: plan to seed a real dev account (`--accountId`)

Status: IMPLEMENTED, pushed on PR #10505. Updated 2026-10-15. Full plan text: `PLAN-SEED-REAL-ACCOUNT-HIGG-43344.md` in the repo root (untracked). The user's dev account is `67a246d8e395ea000c4e11c2`.

## Goal
`ts/bin/seed-account-service.ts --email=<dev> --accountId=<real WCP account id>` seeds the fixture links onto the dev's existing account instead of a created tagged `WCP-A1`.

## Final design
1. Optional `--accountId`; absent = unchanged. Reject ids starting `seed43344-` (another dev's cleanup would delete them).
2. In anchor mode remove `WCP-A1` from `cast.accounts` BEFORE the ES existence check in `seedAccounts`. Otherwise `syncDevRole` would update or delete the dev's membership on the real account. Reject personas with no Alpha membership (`ben`).
3. `seedLinks` takes the anchor as an optional `ISeedAnchor { accountId, organizationId }`; the Alpha WCP side and the `ORG-ALPHA` org resolve to it.
4. `resolveAnchorAccount` (`ts/lib/account-service/seed/seed-anchor.ts`): reads via `accountES.getAccountFieldsByIds` with `users` in `_source`; 404 if missing; 400 unless Accepted and `canManageAccountTeam`; `testAccount` unset = warning only.
5. Projection pre-check: one non-waiting `lookupOrganizations`; not found fails fast with 409.
6. Mixed-mode guard (409 "run --cleanup first"): a tagged `WCP-A1` exists, or a link under the runId has an Alpha side on a different org. ONE runId serves both modes.
7. `--cleanup` ignores `--accountId`; the name-tag guard protects the real account.

## Behaviour facts
- `CodedError` carries the HTTP status in `.code`, not `.status`.
- The real account is untagged, so `cleanupAccount` skips it.
- Earlier seed runs (`seed-HIGG-44769`) put links on the same account. Two had an `scm` side, which is not in `AccountServicePlatform`, and broke `GET /organizations/:id/links` with 500 "Unknown platform: scm". They were hard-deleted on 2026-10-15 (`903badc6…`, `78dd4c96…`).

## Run (dry run first, then `--dryrun=false`)
`source env/dev && HIGG_PROJECTION_GRAPHQL_URL='https://projection-layer.development.higg.org' HIGG_ACCOUNT_SERVICE_BINDING_ORGANIZATIONS=real HIGG_ACCOUNT_SERVICE_BINDING_LINKS=real npx ts-node 'ts/bin/seed-account-service.ts' --email=wellington.lorindo@worldly.io --accountId=67a246d8e395ea000c4e11c2 [--dryrun=false] [--cleanup]`. The URL is the PL root, no `/graphql`. Without the env vars the environment guard and preconditions fail.

## Tests
The seed tests are local-only now, not in the PR: see `procedures/local-only-unit-tests.md` (`ts/test/local/account-service/seed/`, 69 tests).

## State on 2026-10-15
- Branch `HIGG-43344-seed-data`, PR #10505, pushed through `26445e7c4d`. Commits: `364a5a4978` (source), `87e5a3218c` (tests, later removed), `383aa68e84` (README).
- Real seed ran against the account: 5 tagged accounts and 6 links created under scope `a23e42aada`. They are live on dev until `--cleanup --dryrun=false`.
- Read-back as the user worked after the `scm` fix: the org links list returns 200 with 7 links, and details match `ILinkSummary`/`ILinkDetail`.
- Uncommitted, do not commit: `.vscode/launch.json` (env block) and `dist/swagger.json`.
- Not verified: whether PL reads `as_links`; account `users` and `_rev` before and after; an actual confirm or decline.
- Never run plain `npm install`; use `./docs/libxl-troubleshoot/install.sh`.
