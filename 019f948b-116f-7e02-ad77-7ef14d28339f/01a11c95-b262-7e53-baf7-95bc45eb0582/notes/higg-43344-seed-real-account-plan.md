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
  at: 2026-10-14T19:51:27Z
---
# HIGG-43344 seed script: plan to seed a real dev account (`--accountId`)

Status: DRAFT, pending Opus review. No code written. Saved 2026-10-14. Full plan text also at `PLAN-SEED-REAL-ACCOUNT-HIGG-43344.md` in the repo root (untracked).

## Goal
`ts/bin/seed-account-service.ts --email=<dev> --accountId=<real WCP account id>` seeds the fixture link set onto the dev's existing account instead of a tagged `WCP-A1`. The user's dev account is `67a246d8e395ea000c4e11c2` (not yet looked up; read-only lookup was proposed first).

## Design (draft)
- New optional `--accountId`; absent = behaviour unchanged.
- The real account replaces only the Alpha anchor `WCP-A1`. The other five accounts stay tagged counterparts. Four of six links touch ORG-ALPHA, so they land on the real org.
- Real account is read-only: verify it exists (404) and the dev is an Accepted owner/admin on it (400). Never call createEntity, updateEntity or syncDevRole on it.
- One resolver `accountIdFor(target, fixtureAccountId)` in `seed-scope.ts` replaces direct `scopedAccountId` calls in seed-accounts, seed-links (`buildPair`, `orgIdByFixtureOrgId`) and the cleanup loop in the bin script.
- Anchor must already be projected: one non-waiting `lookupOrganizations` before any write; `found: false` fails fast.
- Cleanup: links by `runId`; accounts only when the name carries the seed tag, so the real account can never be deleted. `--cleanup` ignores `--accountId` and logs it.
- Mixed-mode guard: existing seeded links from the other mode under the same runId -> 409 "run --cleanup first" (needs a DAO read by runId in `ts/mongo/as-link-mongo.ts`).
- Environment guard unchanged. Dry run does all reads, no writes.
- Docs: README note; spec edit `docs/specs/HIGG-43344-account-service-seed-script-spec.md` needs the inline modification annotation (date, git user, rationale) per `.claude/rules/agent.md`.

## Tests to write first (TDD, ts/test/account-service/seed/)
accountIdFor; anchor skipped with no create/update/sacId; 404 and 400 role cases; links carry real account and org id on Alpha; cleanup never destroys the untagged real account; mixed-mode 409 both ways; projection pre-check fails fast; dry run writes nothing. Manual: before/after compare of the real account's `users` and `_rev`.

## Open questions sent to Opus (answers NOT yet received)
1. Does the ES account doc expose `users` with role and status, or must the role check read CouchDB?
2. Is "replace only WCP-A1" the right anchor?
3. Hard 400 on non-owner/admin, or a warning?
4. Require `testAccount: true` as a second guard?
5. Is the mixed-mode 409 enough, or should anchor mode use its own runId?
6. Does anything in the link set break on a real org (existing members, live links, real bhive row, PENDING_LINK_STATES)?

TODO next: read the Opus review (a background agent; if lost with the context, re-run a review of the plan file), update the plan, show the user, then implement only after the user approves.

## Known risks to keep
- Links on the real org are visible to everyone reading that org on shared dev; a `proposed` bhive link flips the org profile's bhive row to `pending`.
- This weakens the spec's "tagged accounts only" guarantee to "one real account is read, never written".

## Branch and environment state
- Branch `HIGG-43344-seed-data`, PR #10505 (draft). Pushed commits: `a4a44698f9` (preconditions check resolved PL clients), `086c90b0bd` (README: PL root URL). Push with `git push origin HEAD` (local branch tracks origin/Development).
- Uncommitted, do not commit: `.vscode/launch.json` (env block: ORGANIZATIONS and LINKS bindings real, PL root URL), `dist/swagger.json` (build output). Untracked reports: `SEED-VERIFICATION-HIGG-43344.md`, the plan file.
- Seeded dev data for scope `a23e42aada` was cleaned up (6 links, 6 accounts deleted). PL Organizations persist by design.
- Run command (from repo root): `source env/dev && HIGG_PROJECTION_GRAPHQL_URL='https://projection-layer.development.higg.org' HIGG_ACCOUNT_SERVICE_BINDING_ORGANIZATIONS=real HIGG_ACCOUNT_SERVICE_BINDING_LINKS=real npx ts-node 'ts/bin/seed-account-service.ts' --email=wellington.lorindo@worldly.io [--dryrun=false] [--cleanup]`. The URL must be the PL root with no `/graphql`.
- `node_modules` was reinstalled via `./docs/libxl-troubleshoot/install.sh`; never run plain `npm install`.
