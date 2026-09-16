---
tags:
- account-service
- fixtures
- HIGG-44539
- mock
pinned: true
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-16T18:38:28Z
---
# Account Service mock fixture catalog — invariants and why ALPHA↔BETA can't be collision-free

Non-obvious constraints behind `account-service-fixtures/` (HIGG-44539, PR #10302). Derived
expensively from HIGG-44538/44539 plus epic tickets E1-T1 (HIGG-44764), E4-T4 (HIGG-44782),
E5-T2 (HIGG-44786). The README now documents these, but the *reasoning* below is not in the repo.

## The side-orientation invariant (a real defect, not a style choice)

`ts/managers/account-service/mock/link-mapping.ts` maps `awaitingConfirmationFrom` to **`sideB`
alone** for either-side origins (claim_*, user_asserted). So `sideA` MUST be the
originator/claimant/requester and `sideB` the awaited party.

`createClaim` originally put the org's **anchor** account on `sideA` and the provisioned account on
`sideB` whenever the org already held an Account. Result: both account-holding claim fixtures
awaited *the claimant*. FE would render "Pending approval from scm" (the claimant's own brand-new
account) instead of naming the `wcp` account holder actually being waited on.

Anything that reseats claim link sides must preserve: originator on sideA, awaited on sideB.

## Why org-level same-platform collision screening is infeasible in this catalog

E4-T4 requires "no merge places two same-platform accounts under one org", and says proposal
eligibility screens collisions before a customer sees them. Applied at **org** level (not just at
the two named link sides), ORG-ALPHA is the blocker: it holds `wcp` AND `bhive` linked, so it
collides with every other fixture org holding either.

ALPHA↔BETA fails on every platform assignment, because both orgs hold `wcp` whatever the sides name:

| sideA | sideB | fails on |
| --- | --- | --- |
| ALPHA `wcp` | BETA `bhive` | BETA holds `wcp`; ALPHA holds `bhive` |
| ALPHA `bhive` | BETA `wcp` | ALPHA holds `wcp` |
| ALPHA `bhive` | BETA `scm` | ALPHA holds `scm` (pending) |
| ALPHA `wcp` | BETA `scm` | BETA holds `wcp` |

And ALPHA↔BETA is pinned three times by the confirm-rule personas: `ana` on sideA, `ben` on sideB,
`cam` on both (self-confirm collapse), `dee` on neither (403 path). Making it collision-free means
restructuring ORG-ALPHA or ORG-BETA and re-deriving every persona expectation — a catalog rebuild.

**Decision:** screen per link side; record the org-level gap as a README "Known simplification".
`LINK-ACME-CONSOLIDATION-1` (ORG-ACME `wcp` ↔ ORG-ACME-TRANSIENT `bhive`) is the only pair with no
org-level overlap, i.e. the shape every pair would have without the persona pinning.

## Other invariants worth not breaking

- No same-platform link pair, except `LINK-CLAIM-UNINVITED-SOURCEONLY-1`, whose two sides name the
  **same** account (a claim on an org holding none).
- Every link side resolves to a platform row on the org it names — except that link's `sideB`
  (ORG-THETA holds nothing; that is what source-records-only means).
- A `linked` anchor row carries no `linkId` and may appear on many links (competing proposals are
  legal); a `pending` row carries the live proposal's `linkId`.
- Claim and `user_asserted` links seed `confirmedSides.sideA: true`; matcher/bulk seed neither
  (E1-T1 acceptance, verbatim).
- Every claim carries `accountId`, pending included — created pre-filled at claim time (E5-T2), so
  presence must never be read as "applied". `IFixtureClaim.accountId` is therefore required, and
  `toLiveClaimResource` no longer derives it from `link.sideB`.
- `ffc` appears nowhere as a claim or link platform — "FFC is not linkable this phase" (HIGG-44538,
  verbatim). The mock deliberately does **not** reject `ffc` at runtime: that guard is 44538's
  business logic and would change behaviour FE may already build against.

## Authorization tiers

`linkCapable?: boolean` on `IFixtureUser` (absent = capable) is **user-level, not per-org**, because
HIGG-44538 leaves open whether link-capability means authorized on any account in the org or only on
the accounts a given link touches. Gate lives in `viewerSidesFor`, so a plain member lands on
neither side of any link — that alone denies `GET /links/:id` and forces both viewer affordances
false. Only the org-addressed link routes need `requireLinkCapable`. Persona `vic` is the fixture.
