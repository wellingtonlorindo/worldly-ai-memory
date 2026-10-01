---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-10-01T14:46:13Z
---
# HIGG-44797 org search: state, decisions, open items (2026-10-01)

## Done (draft PR #10440, branch HIGG-44797-organization-search-for-claiming)
- Real `GET /account-service/v1/organizations/search` over PL search. Shared mapper `org-search-result-mapper.ts`, `evaluateOrgEligibility` widened to `OrgEligibility | undefined`, `pendingClaimOrganizationIds` reducer, batched `listByOrganizations` link read.
- **Phase 5 (identifiers) implemented and committed.** `IOrganizationSearchHit` carries nullable `sacId`/`taxId`/`ffcId`. Search enriches hits via one batched `accountNodes(where:{id:{in}})` (new `AS_GetAccountIdentifiers`), skipped when no hit has an account; any failure fails the search. Summary maps the ids from `AS_GetOrganization`. Mapper builds nested optional `identifiers` (strings, nulls omitted, absent when empty). Mock join rows sample `identifiers`. Swagger (`dist` + `dist-account-service-api`) and `ts/routes.ts` regenerated.
- 5 thematic commits: projection layer, graphql+generated, contract+mapper+swagger, fixtures, tests. `npm run build` clean; account-service 564 pass, projection-layer 128 pass.

## Open (unchanged)
- **taxId exposure risk** — stated in the PR, awaiting a mitigation choice (join-only / mask / flag). Owner instruction was to return the ids; implementation follows that.
- **BHive ID** — still no PL source; needs a PL ticket.
- Spec `docs/specs/HIGG-44797-organization-search-for-claiming-spec.md` already annotated with the summary-identifier override (2026-10-01, Wellington Lorindo).

## Gotchas
- Repo commits must NOT carry a `Co-Authored-By` trailer (commit-messages skill), despite the harness attribution reminder.
- PreToolUse hooks require loading `code-comments` before writing comments and `commit-messages` before `git commit`.
- `scriptedFetch` in client.test.ts reuses the last response, so search tests need two scripted responses (REST search then accountNodes). The contract harness (`client-contract.test.ts`) dispatches on `variables.accountIds` vs `variables.organizationId` vs bodyless REST search.
- Untracked scratch file `claiming-matching-be-status.md` is not ours — leave it.
