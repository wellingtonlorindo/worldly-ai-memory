---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-02T18:58:33Z
---
# CI flaky tests & CodeBuild log access (Backend-CI_Testing)

## The CI changedSince pitfall

Backend-CI_Testing runs jest with `--changedSince origin/${baseBranch}` — it only runs test suites for files changed relative to the PR's **base branch**. Consequence: retargeting a PR's base (or moving it off a stacked branch) silently *widens* the changed-file set, so previously-untested suites run for the first time and can surface **pre-existing/flaky failures that are NOT the PR's regression**.

When a CI failure appears right after a base retarget, suspect this before suspecting the PR:
- `git log --oneline <base>..HEAD -- <path>` to see if the failing file was actually touched.
- If neither side touched it, the failure is pre-existing and only newly *in scope*.

## Known flake: ffc-gateway.test.ts

`ts/test/lib/ffc-gateway.test.ts › accountFuzzyMatch › "is name valid and country invalid"` flakes:
```
Expected: false
Received: true   (line ~220)
```
The test draws two independent random countries (`acct1.country = faker.address.country()` vs `ffcApiResponse.country = faker.address.country()`) and asserts they differ. When faker draws the **same country twice**, `matchAccount` sees matching country → `isMatched: true` → fails. Pure collision flake. faker is `@faker-js/faker@7.6.0` (deprecated `name.fullName`/`address.*` still functional).

Durable fix (test-only, outside any given PR's scope): pin distinct fixtures instead of two independent random `country()` draws.

## Separate local-only failure (not CI): elasticsearch-utils date TZ

`unwrapDctReportingPeriod` uses `new Date("2020-01")` → parsed as UTC midnight, then `getFullYear()`/`getMonth()` in **local** TZ. In a TZ west of UTC (e.g. most of the Americas) `"2020-01"` yields `year 2019, month 12`, so `ts/test/es/elasticsearch-utils.test.ts` fails **locally** but passes in CI's TZ. Not a regression — a TZ-dependent pre-existing test.

## Accessing the CodeBuild logs via SSO

- CI project lives in the **staging** account, not the default. Use `--profile sacsecrets-staging` (account 529198457617). Default profile only has `n8n-…`, `development_api-higg-org`, `datacheck`.
- Project name is `Backend-CI_Testing` (underscore). Build IDs are `Backend-CI_Testing:<uuid>`.
- Find builds: `aws codebuild list-builds-for-project --project-name "Backend-CI_Testing" --profile sacsecrets-staging`
- Metadata: `aws codebuild batch-get-builds --ids "Backend-CI_Testing:<uuid>" --profile sacsecrets-staging` (shows `sourceVersion: pr/10440` and CloudWatch log group/stream).
- Logs: `aws logs tail --profile sacsecrets-staging "Backend-CI_Testing" --log-stream-names "<uuid>"` — the failure ("Summary of all failing tests") is near the end.
