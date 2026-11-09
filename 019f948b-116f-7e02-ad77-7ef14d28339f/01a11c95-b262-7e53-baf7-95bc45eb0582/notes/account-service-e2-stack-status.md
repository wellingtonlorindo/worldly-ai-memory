---
kind: project-status
entities:
- higg-44749
- higg-44771
- higg-44772
- higg-44773
- higg-44805
- caseautoapprovalservice
- casemintingservice
- ascasemongo
- asdispositionmongo
tags:
- account-service
- e2
- HIGG-44749
- HIGG-44771
- HIGG-44772
- HIGG-44773
- stacked-prs
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.6.1
  at: 2026-11-09T20:40:22Z
---
# Account Service E2 stack: status and open gaps (2026-11-09)

Epic HIGG-44749 (E2: Case model and minting), under HIGG-43344. All three tickets are "Ready for Integration", assigned to Wellington, and each PR is approved by nmai.

## Stacked PRs (each targets the branch below it)
| Ticket | PR | Branch | Base | Head after 2026-11-09 |
|---|---|---|---|---|
| HIGG-44771 T1 cases + dispositions collections | #10587 | `HIGG-44771-cases-dispositions-collections` | `Development` | `16ce767753` |
| HIGG-44772 T2 case minting service | #10602 | `HIGG-44772-case-minting-service` | T1 branch | `81e52ef2f4` |
| HIGG-44773 T3 auto-approval stopgap | #10621 | `HIGG-44773-auto-approval-stopgap` | T2 branch | `941118f516` |

Each branch has its own git worktree: `api.higg.org-<branch>` next to the main repo. To update the stack: merge Development into T1, then T1 into T2, then T2 into T3. Run `npm run generate-code`, then tsc and `ts/test/account-service` on each, then push. On 2026-11-09 all three were 0 behind Development and mergeable; T1 migration is 2670.

## User rules set this session
- **Jira tickets are the source of truth**; the `docs/specs/*` files can be stale. Fix PRs against the Jira acceptance criteria.
- **Do not post Jira comments** unless asked.
- The Jira CLI token returned 401 on 2026-11-09, so the user said to use the Atlassian MCP (cloudId `worldly.atlassian.net`) for that session.
- When code changes, update the specs too, with a dated annotation under the git user's name.

## Gotchas
- `docs/specs` is a **gitignored symlink** into the main repo `api.higg.org/docs/specs`. Spec edits are local only and never appear in PRs, even though the PR descriptions say "Implements: docs/specs/...".
- `npx tsc --noEmit` runs out of memory (exit 134) under the default heap. Use `NODE_OPTIONS=--max-old-space-size=12288`. Don't pipe it through `tail`, or the exit code is hidden.
- Committing requires the `commit-messages` skill: no `Co-Authored-By` trailer, ticket prefix, one decision per commit. Writing new comments requires the `code-comments` skill (enforced by a hook).

## Fixed on 2026-11-09 (against Jira)
- **T2** `7fe8761126`: `evidenceAtOpen` now stores `pair` and the T9 `candidates`. Before, `candidates` were validated and dropped, which broke "evidence captured at mint time".
- **T2** `81e52ef2f4`: `LinkCaseMinter` takes `triggeredAt` from `link.createdAt`, not mint time. Before, the one-day lag was always about 0.
- **T3** `941118f516`: `LinkHold`/`LinkState` enums replace the string literals; `IAutoApprovalResult` renamed to `AutoApprovalResult`, since the `I` prefix is for interfaces only.
- The #10587 description now says migration 2670 (it said 2631) and 788 tests / 52 suites.

## Open: T3 gaps against HIGG-44773 that need a user or owner decision (owner: than.elliott)
1. **"Full customer path completes end to end"** isn't met. `DeferredLinkApplicationEngine` is a no-op, so the Link stays `confirmed`, the case stays `open`, and the customer sees "pending". Spec Q4 rewords this criterion; owner sign-off is pending and Jira is not updated.
2. **The review set includes never-approved cases.** The disposition is written before the Link is released. If the release fails, an `approve_claim` system disposition remains (`case-auto-approval-service.ts` ~84-94). Fixable: reorder, or filter the review query.
3. **Assertions:** only the flag goes through the port; E3 doesn't exist yet. `recordAssertion` runs only when `transitioned`, so a resumed run never retries it.
4. **Only T3 claims are auto-approved.** Jira scope also names queued merges (T10), standing-signal claims (T8) and in-run bulk approval; the spec drops them.
5. **HIGG-44805 "CS can re-adjudicate every auto-approved case"** can't happen. No transition leaves `confirmed`/`applied`. Once E4 closes cases, D6 refuses a disposition on a closed case and CC3 forbids reopening. E10-T4 needs a Link reversal transition plus a follow-on-case path.

## Known but not Jira criteria (left out on purpose)
- `findByDedupeKey({anyState:true})` scans the whole cases collection, because the `dedupeKey` index is partial on non-closed states.
- Two open cases on the same pair are possible if two mints race with different trigger keys.
- Competing-claim auto-approval can race across claimants, and the check ignores alias/ancestor org ids.
- Promised Jira notes (HIGG-44781, 44792, 44774, 44805) and dropping the stale "Blocked by PL-T11" from HIGG-44749 epic were **not posted** (user said no Jira comments).
