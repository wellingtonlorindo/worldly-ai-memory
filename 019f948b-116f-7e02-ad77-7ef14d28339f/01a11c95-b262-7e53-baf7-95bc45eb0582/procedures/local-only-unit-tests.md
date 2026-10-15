---
kind: procedure
entities:
- ts/test/local
- jest
- seed-account-service
tags:
- testing
- local
- jest
- higg-43344
tier: procedural
type: Procedure
generated:
  by: process:ai-memory/2.6.1
  at: 2026-10-15T18:18:34Z
---
# Local-only unit tests (`ts/test/local/`)

The user keeps unit tests they do not want in a PR under `ts/test/local/`. Your global gitignore (`~/.gitignore_global`, line `local/`) keeps the whole folder out of git, so nothing there is tracked or reaches CI.

## What lives there
- `ts/test/local/bin/import-anonymized-msi-materials/` holds 17 suites and a `mocks.ts` helper (76 tests, all passing on 2026-10-15). They cover `ts/bin/import-anonymized-msi-materials.ts`.
- `ts/test/local/account-service/seed/` holds the HIGG-43344 seed script tests: 8 suites, 69 tests (`await-projection`, `fixture-cast`, `preconditions`, `seed-accounts`, `seed-anchor`, `seed-environment`, `seed-links`, `seed-scope`). Moved out of PR #10505 on 2026-10-15 in commit `26445e7c4d`.

## How to run them
Same command as any test, with the env sourced first and the path quoted:

```
source env/test && npx jest 'ts/test/local'
source env/test && npx jest 'ts/test/local/account-service/seed'
```

Jest still finds them because the ignore only affects git, not the jest config.

## Rules for adding tests here
- Mirror the source path under `ts/test/local/` and add one extra `../` to every relative import, since the files sit one level deeper than `ts/test/`. A copy from `ts/test/account-service/seed/` needs `../../../../` instead of `../../../`.
- Keep `envInit("info")` at the top and no `as any`, as in the tracked tests.
- Do not `git add -f` them. If one should be shared, move it to `ts/test/` and drop the extra `../`.

## Consequences to remember
- CI coverage thresholds do not count these files. The seed code under `ts/lib/account-service/seed/` and `listLinksByRunId` in `ts/mongo/as-link-mongo.ts` have no tracked tests, so a coverage gate can fail on the PR.
- Earlier commits on the branch (for example `87e5a3218c`) still contain the old tracked seed tests. A squash merge hides them; a regular merge keeps them in history.
- Because the folder is untracked, it is not in worktrees. A worktree needs its own copy or a symlink.
