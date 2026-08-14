---
name: Import New Jira Issues
emoji: 📥
description: Create GitHub developer issues for new Jira source items
engine: copilot
model: auto
max-ai-credits: 35
on:
  schedule: every 6h
  workflow_dispatch:
permissions:
  contents: read
  issues: read
  copilot-requests: write
tools:
  github:
    mode: gh-proxy
    toolsets: [issues]
steps:
  - name: Read recent Jira source issues
    env:
      JIRA_BASE_URL: https://joshjohanning.atlassian.net
      JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
      JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
    run: |
      mkdir -p /tmp/gh-aw/agent
      if [ -n "$JIRA_EMAIL" ]; then
        AUTH_ARGS=(--user "$JIRA_EMAIL:$JIRA_TOKEN")
      else
        AUTH_ARGS=(--header "Authorization: Bearer $JIRA_TOKEN")
      fi
      curl --fail-with-body --silent --show-error \
        "${AUTH_ARGS[@]}" \
        --header "Accept: application/json" \
        --get \
        --data-urlencode "jql=project = GHAW ORDER BY created DESC" \
        --data-urlencode "maxResults=10" \
        --data-urlencode "fields=summary,description,issuetype,status,priority,project,assignee,parent,issuelinks,labels,created,updated" \
        "$JIRA_BASE_URL/rest/api/3/search/jql" \
        > /tmp/gh-aw/agent/jira-issues.json
network:
  allowed:
    - defaults
    - joshjohanning.atlassian.net
safe-outputs:
  create-issue:
    labels: [jira-synced]
    max: 3
    deduplicate-by-title: true
---

# Import New Jira Issues

## Task

Jira is the source of truth and GitHub is the developer workspace.

Read the recent Jira items from `/tmp/gh-aw/agent/jira-issues.json`. Use read-only GitHub issue search to find which Jira keys already have GitHub issues in this repository.

For at most three Jira items that do not yet have a GitHub issue, call `create_issue` once per item:

- title: `[<Jira key>] <Jira summary>`
- body:
  - `Jira key: <key>`
  - a `## Jira source` section with key/link, type, status, priority, project, assignee, parent, dependencies, created time, updated time, and a note that Jira owns the section
  - a `## Developer notes` section stating that implementation details, checklists, and links added there are GitHub-owned and preserved during refreshes

Preserve Jira facts. Do not invent values. Use `Not mapped` for unavailable fields.

Do not create duplicates. Call `noop` with a short reason when every Jira item already has a GitHub issue or when the Jira response is invalid.
