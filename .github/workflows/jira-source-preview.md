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
steps:
  - name: Extract Jira key
    id: jira-key
    uses: actions/github-script@v9
    with:
      script: |
        const body = context.payload.issue?.body ?? "";
        const match = body.match(/\bGHAW-\d+\b/i);
        if (!match) {
          core.setFailed("The issue body must contain a GHAW Jira key.");
          return;
        }
        core.setOutput("key", match[0].toUpperCase());
  - name: Read Jira source issue
    env:
      JIRA_BASE_URL: https://joshjohanning.atlassian.net
      JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
      JIRA_KEY: ${{ steps.jira-key.outputs.key }}
      JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
    run: |
      mkdir -p /tmp/gh-aw/agent
      if [ -n "$JIRA_EMAIL" ]; then
        curl --fail-with-body --silent --show-error \
          --user "$JIRA_EMAIL:$JIRA_TOKEN" \
          --header "Accept: application/json" \
          "$JIRA_BASE_URL/rest/api/3/issue/$JIRA_KEY?fields=summary,description,issuetype,status,priority,project,assignee,parent,issuelinks,labels,updated" \
          > /tmp/gh-aw/agent/jira-issue.json
      else
        curl --fail-with-body --silent --show-error \
          --header "Authorization: Bearer $JIRA_TOKEN" \
          --header "Accept: application/json" \
          "$JIRA_BASE_URL/rest/api/3/issue/$JIRA_KEY?fields=summary,description,issuetype,status,priority,project,assignee,parent,issuelinks,labels,updated" \
          > /tmp/gh-aw/agent/jira-issue.json
      fi
network:
  allowed:
    - defaults
    - joshjohanning.atlassian.net
safe-outputs:
  add-comment:
    max: 1
    target: triggering
---

# Jira Source Refresh Preview

## Task

The `preview-jira-refresh` label was applied to a GitHub issue. Act as a work item synchronization planner where Jira is the source of truth and GitHub is the developer workspace.

Read the triggering issue from the sanitized event context. Read the normalized Jira Cloud API response from `/tmp/gh-aw/agent/jira-issue.json`.

Treat these Jira fields as authoritative: summary, issue type, status, priority, project, team, assignee, parent, dependencies, and Jira URL.

Treat developer notes, implementation details, local checklists, links, and issue discussion as GitHub-owned. Do not recommend deleting or overwriting GitHub-owned content.

Create one comment on the triggering issue with:

1. `### Jira source refresh preview`
2. The Jira key, source URL, and a clear statement that Jira remains authoritative.
3. A comparison table with Jira value, current GitHub value, owner, and proposed action for title, type, status, priority, project, team, assignee, parent, and dependencies.
4. `### Proposed GitHub changes` listing only fields that differ.
5. `### Preserved GitHub content` listing developer-owned content that must remain unchanged.
6. A fenced `json` block containing `sourceSystem`, `sourceKey`, `targetSystem`, `authoritativeFields`, `proposedChanges`, and `preservedFields`.
7. `### Integration boundary` stating that this run read Jira Cloud, did not write to Jira, and did not update the GitHub issue.

Preserve facts from both records. Do not invent mappings or IDs. Use `Not mapped` or `null` when unavailable.

Use the `add_comment` safe output exactly once. Call `noop` with a short reason if the Jira response is invalid or no meaningful comparison can be made.
