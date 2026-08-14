---
name: Jira Source Refresh Preview
emoji: 🔄
description: Compare a Jira source record with a GitHub developer work item
engine: copilot
model: auto
on:
  label_command: preview-jira-refresh
permissions:
  contents: read
  issues: read
  copilot-requests: write
tools:
  github:
    mode: gh-proxy
    toolsets: [issues]
safe-outputs:
  add-comment:
    max: 1
    target: triggering
---

# Jira Source Refresh Preview

## Task

The `preview-jira-refresh` label was applied to a GitHub issue. Act as a work item synchronization planner where Jira is the source of truth and GitHub is the developer workspace.

Read the triggering issue from the sanitized event context and find a Jira key matching `[A-Z][A-Z0-9]+-[0-9]+`. Read the corresponding source record from `.demo/jira/<JIRA_KEY>.json`.

Treat these Jira fields as authoritative: summary, issue type, status, priority, project, team, assignee, parent, dependencies, and Jira URL.

Treat developer notes, implementation details, local checklists, links, and issue discussion as GitHub-owned. Do not recommend deleting or overwriting GitHub-owned content.

Create one comment on the triggering issue with:

1. `### Jira source refresh preview`
2. The Jira key, source URL, and a clear statement that Jira remains authoritative.
3. A comparison table with Jira value, current GitHub value, owner, and proposed action for title, type, status, priority, project, team, assignee, parent, and dependencies.
4. `### Proposed GitHub changes` listing only fields that differ.
5. `### Preserved GitHub content` listing developer-owned content that must remain unchanged.
6. A fenced `json` block containing `sourceSystem`, `sourceKey`, `targetSystem`, `authoritativeFields`, `proposedChanges`, and `preservedFields`.
7. `### Integration boundary` stating that this run read a repository fixture, did not call Jira Cloud, and did not update either system.

Preserve facts from both records. Do not invent mappings or IDs. Use `Not mapped` or `null` when unavailable.

Use the `add_comment` safe output exactly once. Call `noop` with a short reason if no Jira key is present, the fixture is missing, the fixture is invalid, or no meaningful comparison can be made.
