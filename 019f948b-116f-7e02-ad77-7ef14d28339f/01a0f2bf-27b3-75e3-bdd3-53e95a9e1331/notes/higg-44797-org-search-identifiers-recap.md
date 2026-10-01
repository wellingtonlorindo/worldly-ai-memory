---
tags:
- HIGG-44797
- account-service
- org-search
- identifiers
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-01T14:23:56Z
---
# HIGG-44797 org search: state, decisions, open items (2026-10-01)

## Done (draft PR #10440, branch HIGG-44797-organization-search-for-claiming)
- Real `GET /account-service/v1/organizations/search` over PL search. Shared mapper `org-search-result-mapper.ts`, `evaluateOrgEligibility` widened to `OrgEligibility | undefined` (no rules; E5-T1/E12-T5 own Retired + type-gate), `pendingClaimOrganizationIds` reducer, batched `listByOrganizations` link read.
- 4 thematic commits, 578 account-service tests pass, `npm run build` clean.
- Mock `hasPendingClaim` now uses `projectLinkStatus === Pending` (confirmed counts too).

## Decisions (owner-confirmed or best judgement)
- Query: trim, cap 600 chars, blank valid (matches mock). No `400 invalid_query` (OQ-3 answer: "respect the mock, keep blank q, trim after 600").
- Extra identifiers: owner said add ALL fields the FE asked for (Worldly account id, SAC id, taxId, ffcId; BHive ID too if possible). Overrides HIGG-44538 principle 3, OQ-30 and the HIGG-44796 "summary exposes no account ids" acceptance.
- Shape: nested optional `identifiers` object on `IOrganizationSearchResult`, all strings, nulls omitted, absent when empty. Not on `attributes` (also backs the member-only profile).
- Sources (verified in ../worldly-projection-layer @66cee55a): PL REST search returns only `worldlyAccountId`. `sacId/taxId/ffcId` live on GraphQL `AccountNode`. Summary: add them to `AS_GetOrganization`. Search: one batched `accountNodes(where:{id:{in}})` call inside `ProjectionLayerClient.searchOrganizations`, skipped when no hit has an account; any failure fails the whole search (no hit without its ids). Wesley (Slack) guessed no PL change is needed; confirmed for sac/tax/ffc.
- BHive ID: NO source today. `OrganizationNode` has no edge to `BhiveAccountNode`, `SmtMemberInfo.bhiveId` is hardcoded null, `OrganizationEntity.bhiveAccountId` is never written. Needs a PL ticket. Do not ship an always-null field.
- Opus 5.5 reviewed the plan: approve with changes (use accountNodes not organizationNodes, strings on the wire, nested object, taxId security flag, existing tests that must be rewritten: mapper key-set test, real/organizations scenario 6, projection-layer client.test.ts and contract.ts).

## Open
- taxId exposure: any authenticated caller + blank q matches whole catalog = tax id harvesting. Options: join-only, mask last 4, ConfigKeysManager flag. Owner has not picked.
- BHive ID needs a PL ticket (populate bhiveAccountId + expose OrganizationNode.bhiveAccount).
- HIGG-44796 spec still needs a dated annotation for the summary-identifier override (not edited).

## State at wrap-up
- Spec `docs/specs/HIGG-44797-organization-search-for-claiming-spec.md` updated (gitignored), incl. new Phase 5 for identifiers.
- Identifier implementation NOT started in production code. Uncommitted leftovers: `ts/graphql/account-service/get-organization.graphql` (+sacId taxId ffcId), new `get-account-identifiers.graphql`, regenerated `ts/graphql/generated/*`, red tests in `ts/test/projection-layer/client.test.ts` (8 failing by design). Revert or continue on user's word.
- Untracked scratch file `claiming-matching-be-status.md` is not ours.

## Gotchas
- Repo commits must NOT carry a Co-Authored-By trailer (commit-messages skill), despite the harness attribution reminder.
- PreToolUse hooks require loading `code-comments` before writing comments and `commit-messages` before `git commit`.
- `scriptedFetch` in client.test.ts reuses the last response, so search tests need two scripted responses once enrichment exists.
