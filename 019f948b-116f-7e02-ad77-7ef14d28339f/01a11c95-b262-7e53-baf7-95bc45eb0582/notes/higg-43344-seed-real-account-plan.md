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
  at: 2026-10-14T19:55:58Z
---
# HIGG-43344 seed script: plan to seed a real dev account (`--accountId`)

Status: REVISED after Opus review (approve with changes). Awaiting user approval. No code written. Updated 2026-10-14. Full text: `PLAN-SEED-REAL-ACCOUNT-HIGG-43344.md` in the repo root (untracked). The user's dev account is `67a246d8e395ea000c4e11c2` (not yet looked up).

## Goal
`ts/bin/seed-account-service.ts --email=<dev> --accountId=<real WCP account id>` seeds the fixture links onto the dev's existing account instead of a created tagged `WCP-A1`.

## Final design
1. Optional `--accountId`; absent = unchanged. Reject ids starting `seed43344-` (another dev's cleanup would delete them).
2. In anchor mode remove `WCP-A1` from `cast.accounts` BEFORE the ES existence check in `seedAccounts`. Otherwise `syncDevRole` would update or delete the dev's membership on the real account (persona `ben` removes it; `ana` changes admin to owner). Reject personas with no Alpha membership.
3. No `ISeedTarget`. `seedLinks` takes WCP side ids and `orgIdByFixtureOrgId` from the `resolved` projection map (it already has `fixtureAccountId` and `accountId`) instead of recomputing `scopedAccountId`.
4. `resolveAnchorAccount`: read via `accountES.getAccountFieldsByIds([id], ["name","users","testAccount"])` (`getPartialsAccountsById` omits `users`); 404 if missing; 400 unless Accepted and `canManageAccountTeam` (`ts/lib/acl/account-user-authority.ts`); `testAccount` unset = warning only.
5. Projection pre-check: one non-waiting `lookupOrganizations`; `found:false` fails fast.
6. Mixed-mode guard (409 "run --cleanup first"): refuse when the scoped tagged `WCP-A1` is in the `bulkFetchByIds` result, or when any link under the runId has an Alpha-org side whose org differs from the anchor's PL org. Keep ONE runId: ACME-CONSOLIDATION-1 and DECLINED-1 share pair keys across modes, so a second runId would hit `LinkConflictError` on the live-pair unique index.
7. Cleanup keeps `scopedAccountId` (it must still delete a leftover tagged WCP-A1); the name-tag guard in `cleanupAccount` protects the real account. `--cleanup` ignores `--accountId` and logs it.
8. Env guard unchanged. Dry run does all reads, no writes.

## Corrections to earlier assumptions
- DECLINE-1 and CONFIRMED-1 touch Alpha only through the synthetic `bhive:BHIVE-A1` side, so their pair keys match in both modes; only MATCHER-1 and APPLIED-1 have a WCP-A1 side.
- The org profile `bhive` row will NOT show pending (`organization-profile-composer.ts:119-123` ignores sides on the profile org's own aliasIds). Expect `ffc` pending via FFC-B1 and a "linked wins" warning on `wcp`. APPLIED-1 makes `link-status-lookup` report the real account as `applied` to a fake bhive account. The earlier README note about the bhive row is wrong for anchor mode. (Opus finding, not re-verified by me.)
- `createLink` then `replaceLink` is not atomic; an abandoned run can leave a `proposed` link on a shared real org until `--cleanup`.

## Tests first (ts/test/account-service/seed/)
Anchor removed from cast (no create/update/sacId, persona `ben` case); `resolveAnchorAccount` 404/400/owner/admin/testAccount warning; `seed43344-` id rejected; links carry real account and org id on Alpha WCP sides only; both 409 guards; cleanup never destroys the untagged real account but still deletes a leftover tagged WCP-A1; projection pre-check fails fast; dry run writes nothing. Manual: compare the account's `users` and `_rev` before and after.

## Open item before the real run
Opus could not verify whether PL reads `as_links` or reacts to new links. Check the PL repo or ask its owner before seeding the real dev account.

## Next steps
1. User approves the revised plan.
2. Implement TDD, small commits, `HIGG-43344:` prefix, no Co-Authored-By trailer (repo `commit-messages` skill). Spec edits need the inline modification annotation. Push with `git push origin HEAD`.
3. Dry run, then real seed on the dev account only with explicit user go-ahead.

## Branch and environment state
- Branch `HIGG-43344-seed-data`, PR #10505 (draft). Pushed: `a4a44698f9` (preconditions check resolved PL clients), `086c90b0bd` (README: PL root URL).
- Uncommitted, do not commit: `.vscode/launch.json` (env block: ORGANIZATIONS and LINKS bindings real, PL root URL), `dist/swagger.json`. Untracked: `SEED-VERIFICATION-HIGG-43344.md`, the plan file.
- Seeded dev data for scope `a23e42aada` is cleaned up. PL Organizations persist by design.
- Run: `source env/dev && HIGG_PROJECTION_GRAPHQL_URL='https://projection-layer.development.higg.org' HIGG_ACCOUNT_SERVICE_BINDING_ORGANIZATIONS=real HIGG_ACCOUNT_SERVICE_BINDING_LINKS=real npx ts-node 'ts/bin/seed-account-service.ts' --email=wellington.lorindo@worldly.io [--dryrun=false] [--cleanup]`. URL is the PL root, no `/graphql`.
- `node_modules` was reinstalled via `./docs/libxl-troubleshoot/install.sh`; never run plain `npm install`.
