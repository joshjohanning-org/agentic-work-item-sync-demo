---
name: Work Item Sync Preview
emoji: 🔄
description: Preview a Digital.ai Agility work item sync when a label is applied
on:
  label_command: sync-to-agility
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

# Work Item Sync Preview

## Task

The `sync-to-agility` label was applied to a GitHub issue. Act as a work item synchronization planner for GitHub Issues and Projects to Digital.ai Agility.

Read the triggering issue from the sanitized event context. If needed, use read-only GitHub tools to inspect its labels, assignees, milestone, parent issue, sub-issues, and project fields.

Create one comment on the triggering issue with a safe, deterministic sync preview. Do not call an external API and do not claim that Digital.ai Agility was changed.

The comment must contain:

1. `### Sync preview`
2. A short create-versus-update recommendation and any external ID found in the issue.
3. A field mapping table covering title, description, type, project, team, status, priority, assignee, parent, and dependencies. Use `Not mapped` when data is unavailable.
4. A fenced `json` block with a normalized payload suitable for a future Digital.ai Agility adapter. Include `operation`, `source`, `externalSystem`, `externalId`, `fields`, and `relationships`.
5. `### Validation` with missing required fields, ambiguous mappings, hierarchy risks, and conflicts. State `No blocking validation issues found.` when none exist.
6. `### Integration boundary` that names the API operation the adapter would perform and states that this run was preview-only.

Preserve facts from the issue. Do not invent IDs, users, teams, projects, statuses, or relationships. Use `null` for unknown JSON values.

Use the `add_comment` safe output exactly once. Call `noop` with a short reason only if the event is not an issue or there is no meaningful issue content to map.
