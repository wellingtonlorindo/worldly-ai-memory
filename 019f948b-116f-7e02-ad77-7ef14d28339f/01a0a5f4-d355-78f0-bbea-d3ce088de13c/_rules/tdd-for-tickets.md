---
tags:
- feedback
- tdd
- process
pinned: true
tier: semantic
type: Rule
generated:
  by: process:ai-memory/2.0.1
  at: 2026-09-15T17:06:51Z
---
# TDD for ticket implementation

When implementing a Jira ticket in this repo (spec doc + `/code`-style flow, or any feature/fixture
work), write the failing test first, then add the minimal implementation/data to make it pass —
red-green-refactor. Use the repo's `tdd` skill explicitly rather than defaulting to
"build the feature, then backfill tests."

**Why:** On HIGG-44539 (Account Service fixture-catalog ticket, worktree
`api.higg.org-HIGG-44539`), I wrote all the new fixture JSON data first (new orgs/users/links/claims),
then wrote one test file asserting against that data afterward — implementation-first, tests-after.
The user interrupted mid-task: "you should TDD your changes for this ticket." The test should have
been written first (naming fixture ids that don't exist yet, confirmed failing) before the fixture
data existed to satisfy it.

**How to apply:** Before adding new fixture data, mock service logic, or DTOs on any ticket: write the
test describing the desired behavior against not-yet-existing fixture ids/state, confirm it fails for
the right reason, then add just enough data/code to turn it green. Do this per scenario, not as one
big data batch followed by one big test batch — each named scenario in a ticket's acceptance criteria
should get its own red→green cycle, and commits should reflect that granularity rather than a single
combined "add fixtures + add tests" commit.
