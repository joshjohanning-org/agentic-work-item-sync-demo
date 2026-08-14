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
        --data-urlencode "fields=summary,description,issuetype,status,priority,project,assignee,parent,issuelinks,labels,created,updated,customfield_10206,customfield_10020,customfield_10208,customfield_10209,customfield_10001,customfield_10210,customfield_10211" \
        "$JIRA_BASE_URL/rest/api/3/search/jql" \
        > /tmp/gh-aw/agent/jira-issues.json
network:
  allowed:
    - defaults
    - joshjohanning.atlassian.net
safe-outputs:
  create-issue:
    labels: [jira-synced]
    allowed-fields: [Priority, Effort]
    max: 3
    deduplicate-by-title: true
  set-issue-field:
    allowed-fields: [Priority, Effort]
    max: 20
    target: "*"
  update-project:
    project: https://github.com/orgs/joshjohanning-org/projects/26
    github-token: ${{ secrets.GH_AW_WRITE_PROJECT_TOKEN }}
    max: 10
---

# Import New Jira Issues

## Task

Jira is the source of truth and GitHub is the developer workspace.

Read the recent Jira items from `/tmp/gh-aw/agent/jira-issues.json`. Use read-only GitHub issue search to find which Jira keys already have GitHub issues in this repository.

For at most three Jira items that do not yet have a GitHub issue, call `create_issue` once per item:

- title: `[<Jira key>] <Jira summary>`
- body:
  - `Jira key: <key>`
  - `<!-- gh-aw-island-start:jira-sync-on-command -->`
  - a `## Jira source` section with key/link, type, status, priority, project, assignee, parent, dependencies, created time, updated time, and a note that Jira owns the section
  - `<!-- gh-aw-island-end:jira-sync-on-command -->`
  - a `## Developer notes` section stating that implementation details, checklists, and links added there are GitHub-owned and preserved during refreshes
- fields:
  - `Priority`: Jira priority, when provided
  - `Effort`: map Jira Story Pts 1-2 → Low, 3-5 → Medium, 8 or more → High

For Jira items that already have a GitHub issue:

1. Use `set_issue_field` to refresh Priority and Effort with the same mappings.
2. Use `update_project` with:
   - `project`: `https://github.com/orgs/joshjohanning-org/projects/26`
   - `content_type`: `issue`
   - `content_number`: the existing GitHub issue number
   - `fields` containing only values available from Jira:
     - `Status`: Jira Backlog, Open, or To Do → `Todo`; In Progress or Selected for Development → `In progress`; Done, Closed, or Resolved → `Done`
     - `PI`: Jira PI
     - `Iteration`: Jira Sprint title
     - `Source`: Jira Source, meaning work origin
     - `Team`: Jira Team
     - `Requested by`: Jira Requested by
     - `Product category`: Jira Product category

Preserve Jira facts. Do not invent values. Use `Not mapped` for unavailable fields.

Do not create duplicates. Newly created issues receive project fields on the next scheduled or manually dispatched run. Call `noop` only when the Jira response is invalid or no create/update action is needed.
