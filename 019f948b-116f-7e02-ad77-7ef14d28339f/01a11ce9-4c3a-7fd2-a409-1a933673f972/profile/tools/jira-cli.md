---
kind: rule
pinned: true
tier: semantic
type: Note
generated:
  by: process:ai-memory/2.6.1
  at: 2026-10-13T16:42:26Z
---
# Use Jira CLI for Jira

Use the Jira CLI (`jira`, ankitpokhrel/jira-cli) for all Jira reads and writes. Do not use the Atlassian MCP server.

- View: `jira issue view HIGG-123 --plain`
- Children: `jira issue list -q 'parent = HIGG-123' --plain --columns key,type,status,assignee,summary`
- Logged in as the user's worldly.io account; site worldly.atlassian.net.

Set by the user on 2026-10-13.