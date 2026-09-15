---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-15T17:16:31Z
---
# HIGG-44539 fixture catalog — TDD rework complete, ready for commit

Worktree `/Users/wellingtonlorindo/Documents/code/worldly/ai/api.higg.org-HIGG-44539`, branch
`HIGG-44539` (off `Development`). Jira HIGG-44539 \"E0-T2: Mock customer-facing routes and fixture
catalog\". Sibling HIGG-44538 (router/types/mock services/registry scaffold) already merged (PR
#10269) — not in scope here. Governing spec:
`../api.higg.org/docs/specs/HIGG-44537-account-service-e0-fe-facing-mock-and-route-contract-spec.md`
(main checkout, not this worktree).

## TDD rework — done

Per `[[tdd-for-tickets]]`, redid all 6 session-added scenarios in proper red→green order (temporarily
removed each new fixture entry, ran the targeted test with `-t` to confirm it failed for the right
reason — \"Unknown link id\"/\"Unknown claim id\"/empty array — then restored the fixture and confirmed
green):

1. Declined pair — `LINK-DECLINED-1`
2. Uninvited claim, target org holds an Account — `CLAIM-UNINVITED-HOLDS-1`
3. Uninvited claim, source-records-only org — `CLAIM-UNINVITED-SOURCEONLY-1`
4. Multi-site disambiguation — `ORG-ZETA-1`/`ORG-ZETA-2` (persona `fin`)
5. User-asserted internal review — `LINK-USERASSERTED-REVIEW-1`
6. User-asserted counterparty approval — `LINK-USERASSERTED-PENDING-1`

The other ~14 scenarios in the test file assert against fixtures that already existed before this
session (HIGG-44538's seed set — `ORG-ALPHA/BETA/GAMMA/DELTA`, `LINK-MATCHER-1`, `LINK-DECLINE-1`,
`LINK-APPLIED-1`, `CLAIM-PENDING-1`, etc.) — those didn't need red/green since the underlying
behavior already existed and worked; they're just first-time test coverage of pre-existing fixtures.

**Verified after rework:** `git status --short` on `account-service-fixtures/` and
`ts/test/account-service/` matches the pre-rework file set exactly (no stray diffs from the
toggling). `npx jest 'ts/test/account-service'` → 13 suites / 143 tests passing. `npx eslint` clean
on both touched `.ts` files. Full `npm run build` kicked off to reconfirm — check its result before
committing if not yet seen.

**No git commit made yet. No PR opened yet.**

## Remaining work

1. If the pending `npm run build` run surfaces unrelated generated-artifact drift (swagger.json /
   ts/routes.ts — this happened once already this ticket, see below), revert those, keep only the
   account-service-fixtures/ + test files.
2. Small, thematically-grouped commits per `.claude/rules/code-quality.md` (ticket-prefixed
   `HIGG-44539: ...`, never `git add -A`).
3. Fable-model subagent (`Agent` tool, `model: \"fable\"`) reviews the changes — user asked for this
   explicitly, to run before/alongside opening the PR.
4. Open a draft PR when done (per the `/draft-pr` command referenced in the spec).
5. Never edit or comment on the Jira ticket itself (explicit instruction).

## Environment setup already done (persists on disk, don't redo)

- `node_modules` installed via `./docs/libxl-troubleshoot/install.sh` (never raw `npm install` — see
  repo CLAUDE.md, needs Python 2.7 via pyenv for the `libxl` native module).
- macOS libxl dylib fix already applied: `install_name_tool -change libxl.dylib
  @loader_path/../../deps/libxl/lib/libxl.dylib node_modules/libxl/build/Release/libxl.node` —
  without it every jest run fails with \"unable to load libxl.node\".
- `env/test` created locally from `env/.test-template` (gitignored, safe to recreate) with placeholder
  CouchDB credentials — account-service router tests never hit real `DataServices`/CouchDB. Run
  tests with: `source env/test && npx jest 'ts/test/account-service'`.
- Previously (earlier session), `npm run build` regenerated 3 unrelated pre-existing drift files
  (`dist-account-service-api/swagger.json`, `dist-corporate-report-api/swagger.json`,
  `dist/swagger.json`, `ts/routes.ts` — PIC/MatLib `activityName` staleness already on `Development`,
  unrelated to this ticket) — these were reverted and must not be committed.
