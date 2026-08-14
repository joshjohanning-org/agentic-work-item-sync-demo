---
name: Sync GitHub Issue from Jira
emoji: 🔄
description: Refresh Jira-owned fields while preserving GitHub developer content
engine: copilot
model: auto
max-ai-credits: 25
on:
  label_command: sync-from-jira
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
          "$JIRA_BASE_URL/rest/api/3/issue/$JIRA_KEY?fields=summary,description,issuetype,status,priority,project,assignee,parent,issuelinks,labels,updated,customfield_10206,customfield_10020,customfield_10208,customfield_10209,customfield_10001,customfield_10210,customfield_10211" \
          > /tmp/gh-aw/agent/jira-issue.json
      else
        curl --fail-with-body --silent --show-error \
          --header "Authorization: Bearer $JIRA_TOKEN" \
          --header "Accept: application/json" \
          "$JIRA_BASE_URL/rest/api/3/issue/$JIRA_KEY?fields=summary,description,issuetype,status,priority,project,assignee,parent,issuelinks,labels,updated,customfield_10206,customfield_10020,customfield_10208,customfield_10209,customfield_10001,customfield_10210,customfield_10211" \
          > /tmp/gh-aw/agent/jira-issue.json
      fi
network:
  allowed:
    - defaults
    - joshjohanning.atlassian.net
safe-outputs:
  update-issue:
    title:
    body:
    max: 1
    target: "*"
  add-labels:
    allowed: [jira-synced]
    max: 1
    target: triggering
  set-issue-field:
    allowed-fields: [Priority, Effort]
    max: 2
    target: "*"
---

# Sync GitHub Issue from Jira

## Task

The `sync-from-jira` label was applied to a GitHub issue. Jira is the source of truth and GitHub is the developer workspace.

Read the triggering issue from the sanitized event context. Read the normalized Jira Cloud API response from `/tmp/gh-aw/agent/jira-issue.json`.

Treat these Jira fields as authoritative: key, summary, issue type, status, priority, project, assignee, parent, issue links, labels, updated time, and Jira URL.

Treat developer notes, implementation details, local checklists, links, and issue discussion as GitHub-owned. Preserve them exactly.

Use `update_issue` exactly once and set `issue_number` to the triggering GitHub issue number:

1. Set the title to `[<Jira key>] <Jira summary>`.
2. Set `operation` to `replace-island`.
3. Set the body to a concise `## Jira source` section containing:
   - Jira key and link
   - issue type
   - status
   - priority
   - project
   - assignee
   - parent
   - issue links or dependencies
   - Jira updated timestamp
   - a note that Jira owns this section

The replace-island operation must only replace this workflow's managed section. Do not include or rewrite GitHub-owned developer content in the update body.

Use `add_labels` to add `jira-synced`.

Use `set_issue_field` for the triggering issue:

- Set `Priority` from Jira priority when Jira provides one.
- Set `Effort` from Jira Story Pts: 1-2 → Low, 3-5 → Medium, 8 or more → High. Omit Effort when Story Pts is unavailable.

Do not write to Jira. Do not invent mappings or IDs. Use `Not mapped` when Jira does not provide a value. Call `noop` with a short reason if the Jira response is invalid.
