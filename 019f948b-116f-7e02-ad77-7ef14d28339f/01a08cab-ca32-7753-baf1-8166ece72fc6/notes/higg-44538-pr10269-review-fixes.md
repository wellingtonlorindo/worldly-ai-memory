---
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-14T21:19:38Z
---
# HIGG-44538 / PR #10269 — Fable review findings (2026-09-14, RESOLVED)

Context: branch HIGG-44538, PR #10269 (Account Service E0 mock surface).

Update 2026-09-14 (later session): all items below were triaged and fixed, then committed as 9
thematic commits (`968d55dafb`..`8ad43f86f9`) via the `commit-messages` skill, pushed, and the PR
description updated with a "Review-round hardening" section + bumped test counts (113/113, 11
suites). Full verification: `tsc` clean, eslint clean, swagger/routes regenerated and diff-checked
(only the expected response-code/shape changes, no path add/remove).

Fixed:
1. `GET /organizations/resolve` gate no-op — split `AccountServiceContext.from` into
   `identityOnly` (no gate) + `requireEnabledFor` (deferred gate), so the org-scoped canary check
   now has a real org id to scope against. Router calls identity → lookup → gate, in that order.
2. `under_review` claims now carry `hold: LinkHold.InternalReview` (claims.ts createClaim), so the
   claimant can no longer self-confirm/decline out of CS review (sideA===sideB self-claim case).
3. `LINK-CONFIRMED-1` fixture inconsistency — added a matching `pending` wcp row on ORG-GAMMA in
   organizations.json referencing the link, plus a router-level test on the confirmed/queued-merge
   state (viewerCanConfirm/viewerCanDecline false, awaitingConfirmationFrom []).
4. Misleading "same-platform guard" comment in claims.ts corrected (no such guard exists).
6. fault-injection-middleware.ts's bare `res.sendStatus(401)` now goes through `next(CodedError)`
   so both auth-rejection paths (`handleMockReset`, `injectFault`) match the shared error envelope.

Not fixed (accepted as-is, low priority per the original review, not revisited):
- #3 (awaitingConfirmationFrom "sideA is accepted originator" docstring wording for claims) and the
  remaining nits (LINK-UNAUTHORIZED-1 cross-org claim fixture, contracts-d.ts 403 JSDoc line,
  decline rejection message wording, link-mapping.test.ts extra case, fault-injection ordering test,
  LinkHold docstring narrating removed LinkPendingWith).

See [[higg-44538-fable-review-process]] if that gets written up — the pattern (self-review →
Copilot ×2 → spec reconciliation → adversarial Fable subagent pass before commit) is worth
recording as a project convention if it recurs on future tickets.
