---
tags:
- project
- higg-44539
- account-service
- in-progress
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-15T17:07:15Z
---
# HIGG-44539 fixture catalog — in-progress state

Worktree `/Users/wellingtonlorindo/Documents/code/worldly/ai/api.higg.org-HIGG-44539`, branch
`HIGG-44539` (off `Development`). Jira HIGG-44539 "E0-T2: Mock customer-facing routes and fixture
catalog", status Not Started as of 2026-09-15, no comments. Sibling HIGG-44538 (router/types/mock
services/registry scaffold) is already merged (PR #10269) — not in scope here. Governing spec:
`../api.higg.org/docs/specs/HIGG-44537-account-service-e0-fe-facing-mock-and-route-contract-spec.md`
(main checkout, not this worktree).

**Process correction mid-session:** user requires TDD (red-green-refactor) for this ticket's
remaining work — see the `tdd-for-tickets` rule page. What's described below as "already done" was
built implementation-first (fixtures, then one test file asserting against them) and needs to be
reworked/re-justified scenario-by-scenario in proper red→green order before committing.

## Pre-existing state (HIGG-44538's "representative" seed set, before this session)

`account-service-fixtures/` (repo root, outside `ts/`): 7 JSON files (`users.json`,
`organizations.json`, `links.json`, `claims.json`, `organization-search.json`,
`link-candidates.json`, `platform-account-search.json`) + `README.md` saying explicitly: "This is a
representative seed set ... not the exhaustive per-Figma-state catalog HIGG-44539 owns." 4 orgs
(ORG-ALPHA/BETA/GAMMA/DELTA), 4 personas (ana/ben/cam/dee), 7 links. All 15 router endpoints, both
config keys (`EnableAccountService`, `EnableAccountServiceAutoApproval`), and the full mock
service/registry/confirm-rule/link-mapping layer already existed and worked — this ticket is data +
a catalog doc only, no new business logic.

## What this session added (uncommitted, working-tree only — needs TDD rework)

1. Fixture data:
   - `users.json`: personas `fin` (member of new `ORG-ZETA-1`+`ORG-ZETA-2`), `gus` (member of new
     `ORG-ETA`), `theo` (member of new `ORG-THETA`).
   - `organizations.json`: added `ORG-ZETA-1`/`ORG-ZETA-2` (same name "Zeta Apparel Group", different
     orgId/country — multi-site disambiguation scenario), `ORG-ETA` (counterparty org for
     user-asserted flows), `ORG-THETA` (second source-only org, CS-review-held claim). Added a `ffc`
     "pending" row to existing `ORG-BETA` for `LINK-CLAIM-UNINVITED-HOLDS-1`.
   - `links.json`: added `LINK-DECLINED-1` (state declined — previously-missing "declined pair"
     scenario), `LINK-USERASSERTED-REVIEW-1` (user_asserted, hold internal_review, ana↔ORG-ETA/gus),
     `LINK-USERASSERTED-PENDING-1` (user_asserted, hold cleared — "counterparty approval" flavor,
     ben↔ORG-ETA/gus), `LINK-CLAIM-UNINVITED-HOLDS-1` (uninvited claim, org holds account → ordinary
     proposal), `LINK-CLAIM-UNINVITED-SOURCEONLY-1` (uninvited claim, source-only ORG-THETA → CS
     review hold, sideA===sideB===theo).
   - `claims.json`: added `CLAIM-UNINVITED-HOLDS-1`, `CLAIM-UNINVITED-SOURCEONLY-1`.
   - `organization-search.json`: added `ORG-THETA` entry (needed for its `.../summary` route).
   - `README.md`: fully rewritten into the ticket's actual deliverable — full tables per
     org/user/link/claim + a "named scenario → fixture" cross-reference mapping every bullet of the
     Jira ticket's "Named scenarios" list to concrete fixture ids. This doc content documents *data*,
     not test structure — likely still correct/reusable after the TDD rework.
2. `ts/test/account-service/account-service-v1-router/helpers.ts`: added exports `ORG_DELTA`,
   `ORG_ZETA_1`, `ORG_ZETA_2`, `ORG_ETA`, `ORG_THETA`, `USER_FIN`, `USER_GUS`, `USER_THEO`, extended
   `PERSONA_BY_AUTH_USER_ID`.
3. One new test file (written AFTER the data — the part to redo):
   `ts/test/account-service/account-service-v1-router/fixture-catalog-scenarios.test.ts`, ~20 `it`s,
   one per named scenario.

**Verified before the interrupt:** `npx eslint` clean on both touched `.ts` files; `npx jest
'ts/test/account-service'` → 13 suites / 143 tests passing; full `npm run build` (codegen+tsc+clean)
→ exit 0. Reverted 3 unrelated pre-existing generated-artifact drift files that `npm run build`
happened to regenerate (`dist-account-service-api/swagger.json`, `dist-corporate-report-api/swagger.json`,
`dist/swagger.json`, `ts/routes.ts` — PIC/MatLib `activityName` staleness already on `Development`,
unrelated to this ticket) — do not commit those.

**No git commit made yet. No PR opened yet.** `git status --short` at interrupt time: modified
`account-service-fixtures/{README.md,claims.json,links.json,organization-search.json,
organizations.json,users.json}`, modified
`ts/test/account-service/account-service-v1-router/helpers.ts`, untracked
`ts/test/account-service/account-service-v1-router/fixture-catalog-scenarios.test.ts`.

## Environment setup already done (persists on disk, don't redo)

- `node_modules` installed via `./docs/libxl-troubleshoot/install.sh` (never raw `npm install` — see
  repo CLAUDE.md, needs Python 2.7 via pyenv for the `libxl` native module).
- macOS libxl dylib fix already applied: `install_name_tool -change libxl.dylib
  @loader_path/../../deps/libxl/lib/libxl.dylib node_modules/libxl/build/Release/libxl.node` —
  without it every jest run fails with "unable to load libxl.node" (pulled in transitively via
  `ts/managers/config-keys.ts` → `ts/lib/data-services.ts` → ... → `libxl`).
- `env/test` created locally from `env/.test-template` (gitignored, safe to recreate) with placeholder
  CouchDB credentials — account-service router tests never hit real `DataServices`/CouchDB, so
  placeholders are fine. Run tests with: `source env/test && npx jest 'ts/test/account-service'`.

## Remaining work after TDD rework

- Small, thematically-grouped commits per `.claude/rules/code-quality.md` (ticket-prefixed
  `HIGG-44539: ...`, never `git add -A`).
- Open a draft PR when done (per the `/draft-pr` command referenced in the spec).
- User asked to have a Fable-model subagent (`Agent` tool, `model: "fable"`) review the changes once
  done, before/alongside opening the PR.
- Never edit or comment on the Jira ticket itself (explicit instruction).
