---
name: Sync Jira and GitHub Work Item
emoji: 🔄
description: Synchronize a linked work item in the direction selected by its command label
engine: copilot
model: auto
max-ai-credits: 30
on:
  issues:
    types: [labeled]
    names: [sync-from-jira, sync-to-jira]
    lock-for-agent: true
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
        const fs = require("fs");
        const body = context.payload.issue?.body ?? "";
        const match = body.match(/\bGHAW-\d+\b/i);
        if (!match) {
          core.setFailed("The issue body must contain a GHAW Jira key.");
          return;
        }
        core.setOutput("key", match[0].toUpperCase());
        core.setOutput("direction", context.payload.label?.name ?? "");
        const labels = (context.payload.issue?.labels ?? []).map((label) => label.name);
        core.setOutput(
          "conflict",
          String(labels.includes("sync-from-jira") && labels.includes("sync-to-jira"))
        );
        fs.mkdirSync("/tmp/gh-aw/agent", { recursive: true });
        fs.writeFileSync(
          "/tmp/gh-aw/agent/sync-command.json",
          JSON.stringify({
            direction: context.payload.label?.name ?? "",
            conflict: labels.includes("sync-from-jira") && labels.includes("sync-to-jira"),
          })
        );
  - name: Read Jira source issue
    env:
      JIRA_BASE_URL: https://joshjohanning.atlassian.net
      JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
      JIRA_KEY: ${{ steps.jira-key.outputs.key }}
      JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
    run: |
      set -euo pipefail
      mkdir -p /tmp/gh-aw/agent
      if [ -n "$JIRA_EMAIL" ]; then
        AUTH_ARGS=(--user "$JIRA_EMAIL:$JIRA_TOKEN")
      else
        AUTH_ARGS=(--header "Authorization: ******")
      fi
      curl --fail-with-body --silent --show-error \
        "${AUTH_ARGS[@]}" \
        --header "Accept: application/json" \
        "$JIRA_BASE_URL/rest/api/3/issue/$JIRA_KEY?fields=summary,description,issuetype,status,priority,project,assignee,parent,issuelinks,labels,updated,customfield_10218,customfield_10020,customfield_10222,customfield_10221,customfield_10001,customfield_10220,customfield_10219" \
        > /tmp/gh-aw/agent/jira-issue.json
  - name: Generate Project read token
    id: project-read-token
    uses: actions/create-github-app-token@v3
    with:
      client-id: ${{ vars.PROJECT_APP_CLIENT_ID }}
      private-key: ${{ secrets.PROJECT_APP_PRIVATE_KEY }}
      owner: joshjohanning-org
      repositories: agentic-work-item-sync-demo
      permission-issues: read
      permission-organization-projects: read
  - name: Read GitHub issue and Project fields
    env:
      GH_TOKEN: ${{ steps.project-read-token.outputs.token }}
      ISSUE_NUMBER: ${{ github.event.issue.number }}
    shell: bash
    run: |
      set -euo pipefail
      gh api "repos/joshjohanning-org/agentic-work-item-sync-demo/issues/$ISSUE_NUMBER" \
        > /tmp/gh-aw/agent/github-issue.json

      gh api graphql \
        --header "GraphQL-Features: issue_fields" \
        -F owner=joshjohanning-org \
        -F repo=agentic-work-item-sync-demo \
        -F number="$ISSUE_NUMBER" \
        -f query='
          query($owner: String!, $repo: String!, $number: Int!) {
            repository(owner: $owner, name: $repo) {
              issue(number: $number) {
                issueFieldValues(first: 20) {
                  nodes {
                    ... on IssueFieldNumberValue {
                      value
                      field { ... on IssueFieldNumber { name } }
                    }
                    ... on IssueFieldTextValue {
                      value
                      field { ... on IssueFieldText { name } }
                    }
                    ... on IssueFieldDateValue {
                      value
                      field { ... on IssueFieldDate { name } }
                    }
                    ... on IssueFieldSingleSelectValue {
                      name
                      field { ... on IssueFieldSingleSelect { name } }
                    }
                  }
                }
              }
            }
          }' \
        > /tmp/gh-aw/agent/github-issue-fields.json

      gh project item-list 26 \
        --owner joshjohanning-org \
        --format json \
        --limit 100 \
        | jq --argjson number "$ISSUE_NUMBER" \
          '{items: [.items[] | select(.content.number == $number)]}' \
        > /tmp/gh-aw/agent/github-project-item.json
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
  add-comment:
    max: 1
    target: triggering
  add-labels:
    allowed: [jira-synced]
    max: 1
    target: triggering
  remove-labels:
    allowed: [sync-from-jira, sync-to-jira]
    max: 2
    target: triggering
  set-issue-field:
    allowed-fields: [Priority, Effort]
    max: 2
    target: "*"
  update-project:
    project: https://github.com/orgs/joshjohanning-org/projects/26
    max: 1
    github-app:
      client-id: ${{ vars.PROJECT_APP_CLIENT_ID }}
      private-key: ${{ secrets.PROJECT_APP_PRIVATE_KEY }}
      owner: joshjohanning-org
      repositories: [agentic-work-item-sync-demo]
  jobs:
    update-jira:
      description: Update the linked Jira issue from explicit GitHub issue and Project field values
      runs-on: ubuntu-latest
      needs: unlock
      output: Jira work item updated from GitHub
      permissions:
        contents: read
        issues: write
      inputs:
        issue_number:
          description: Triggering GitHub issue number
          required: true
          type: string
        key:
          description: Jira issue key
          required: true
          type: string
        summary:
          description: Jira summary without the bracketed Jira key
          required: true
          type: string
        status:
          description: GitHub Project Status
          required: false
          type: string
        priority:
          description: GitHub Priority issue field
          required: false
          type: string
        effort:
          description: GitHub Effort issue field
          required: false
          type: string
        pi:
          description: GitHub Project PI
          required: false
          type: string
        iteration:
          description: GitHub Project Iteration title
          required: false
          type: string
        source:
          description: GitHub Project Source
          required: false
          type: string
        team:
          description: GitHub Project Team
          required: false
          type: string
        requested_by:
          description: GitHub Project Requested by
          required: false
          type: string
        product_category:
          description: GitHub Project Product category
          required: false
          type: string
      steps:
        - name: Apply GitHub values to Jira
          uses: actions/github-script@v9
          env:
            JIRA_BASE_URL: https://joshjohanning.atlassian.net
            JIRA_BOARD_ID: "3"
            JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
            JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
          with:
            script: |
              const fs = require("fs");

              const outputFile = process.env.GH_AW_AGENT_OUTPUT;
              const jiraBaseUrl = process.env.JIRA_BASE_URL;
              const jiraEmail = process.env.JIRA_EMAIL;
              const jiraToken = process.env.JIRA_TOKEN;
              const boardId = process.env.JIRA_BOARD_ID;

              if (!outputFile || !fs.existsSync(outputFile)) {
                core.setFailed("Agent output was not available.");
                return;
              }
              if (!jiraEmail || !jiraToken) {
                core.setFailed("JIRA_EMAIL and JIRA_TOKEN must be configured.");
                return;
              }

              const auth = Buffer.from(`${jiraEmail}:${jiraToken}`).toString("base64");
              const jiraRequest = async (path, options = {}) => {
                const response = await fetch(`${jiraBaseUrl}${path}`, {
                  ...options,
                  headers: {
                    Authorization: `Basic ${auth}`,
                    Accept: "application/json",
                    "Content-Type": "application/json",
                    ...(options.headers || {}),
                  },
                });
                if (!response.ok) {
                  throw new Error(`Jira ${options.method || "GET"} ${path} failed (${response.status}): ${await response.text()}`);
                }
                if (response.status === 204) return null;
                return response.json();
              };

              const agentOutput = JSON.parse(fs.readFileSync(outputFile, "utf8"));
              const items = agentOutput.items.filter((item) => item.type === "update_jira");
              if (items.length !== 1) {
                core.setFailed(`Expected exactly one update_jira request, received ${items.length}.`);
                return;
              }

              for (const item of items) {
                if (!/^GHAW-\d+$/.test(item.key)) {
                  core.setFailed(`Invalid Jira key: ${item.key}`);
                  return;
                }
                const issueNumber = Number(item.issue_number);
                if (!Number.isInteger(issueNumber) || issueNumber < 1) {
                  core.setFailed(`Invalid GitHub issue number: ${item.issue_number}`);
                  return;
                }

                try {
                  const editMeta = await jiraRequest(`/rest/api/3/issue/${item.key}/editmeta`);
                  const editable = editMeta.fields || {};
                  const fields = { summary: item.summary.trim() };
                  const updated = ["Summary"];
                  const skipped = [];

                  const setOption = (fieldId, value, label) => {
                    if (!value || value === "Not mapped") return;
                    const field = editable[fieldId];
                    if (!field) {
                      skipped.push(`${label} is not editable in Jira`);
                      return;
                    }
                    const option = (field.allowedValues || []).find((candidate) =>
                      [candidate.value, candidate.name].some((name) =>
                        typeof name === "string" && name.toLowerCase() === value.toLowerCase()
                      )
                    );
                    if (!option) {
                      skipped.push(`${label} value "${value}" has no exact Jira option`);
                      return;
                    }
                    fields[fieldId] = { id: option.id };
                    updated.push(label);
                  };

                  setOption("priority", item.priority, "Priority");
                  setOption("customfield_10218", item.pi, "PI");
                  setOption("customfield_10221", item.source, "Source");
                  setOption("customfield_10001", item.team, "Team");
                  setOption("customfield_10219", item.product_category, "Product category");

                  if (item.requested_by && item.requested_by !== "Not mapped") {
                    if (editable.customfield_10220) {
                      fields.customfield_10220 = item.requested_by;
                      updated.push("Requested by");
                    } else {
                      skipped.push("Requested by is not editable in Jira");
                    }
                  }

                  if (item.effort && item.effort !== "Not mapped") {
                    if (editable.customfield_10222) {
                      const pointsByEffort = { low: 2, medium: 5, high: 8 };
                      const points = pointsByEffort[item.effort.toLowerCase()];
                      if (points) {
                        fields.customfield_10222 = points;
                        updated.push("Story pts");
                      } else {
                        skipped.push(`Effort value "${item.effort}" has no Story pts mapping`);
                      }
                    } else {
                      skipped.push("Story pts is not editable in Jira");
                    }
                  }

                  await jiraRequest(`/rest/api/3/issue/${item.key}`, {
                    method: "PUT",
                    body: JSON.stringify({ fields }),
                  });

                  if (item.status && item.status !== "Not mapped") {
                    const transitions = await jiraRequest(`/rest/api/3/issue/${item.key}/transitions`);
                    const desiredNames = {
                      todo: ["to do", "backlog", "open"],
                      "in progress": ["in progress", "selected for development"],
                      done: ["done", "closed", "resolved"],
                    }[item.status.toLowerCase()] || [];
                    const transition = transitions.transitions.find((candidate) =>
                      desiredNames.includes(candidate.to.name.toLowerCase())
                    );
                    if (transition) {
                      await jiraRequest(`/rest/api/3/issue/${item.key}/transitions`, {
                        method: "POST",
                        body: JSON.stringify({ transition: { id: transition.id } }),
                      });
                      updated.push("Status");
                    } else if (desiredNames.length) {
                      skipped.push(`Status "${item.status}" has no available Jira transition`);
                    }
                  }

                  if (item.iteration && item.iteration !== "Not mapped") {
                    try {
                      const sprintData = await jiraRequest(`/rest/agile/1.0/board/${boardId}/sprint?state=active,future`);
                      const sprint = (sprintData.values || []).find(
                        (candidate) => candidate.name.toLowerCase() === item.iteration.toLowerCase()
                      );
                      if (sprint) {
                        await jiraRequest(`/rest/agile/1.0/sprint/${sprint.id}/issue`, {
                          method: "POST",
                          body: JSON.stringify({ issues: [item.key] }),
                        });
                        updated.push("Sprint");
                      } else {
                        skipped.push(`Iteration "${item.iteration}" has no exact active or future Jira sprint`);
                      }
                    } catch (error) {
                      skipped.push(`Sprint was not updated: ${error.message}`);
                    }
                  }

                  const warningText = skipped.length
                    ? `\n\n**Not updated:** ${skipped.join("; ")}.`
                    : "";
                  await github.rest.issues.createComment({
                    owner: context.repo.owner,
                    repo: context.repo.repo,
                    issue_number: issueNumber,
                    body: `## Jira sync complete\n\nUpdated **${item.key}** from GitHub: ${updated.join(", ")}.${warningText}`,
                  });
                  await github.rest.issues.removeLabel({
                    owner: context.repo.owner,
                    repo: context.repo.repo,
                    issue_number: issueNumber,
                    name: "sync-to-jira",
                  });
                } catch (error) {
                  core.setFailed(error.message);
                  return;
                }
              }
---

# Synchronize Jira and GitHub

## Command

Read the triggering command label and trusted conflict check from `/tmp/gh-aw/agent/sync-command.json`.

Jira issue data is in `/tmp/gh-aw/agent/jira-issue.json`. The current GitHub issue, organization issue fields, and Project item are in:

- `/tmp/gh-aw/agent/github-issue.json`
- `/tmp/gh-aw/agent/github-issue-fields.json`
- `/tmp/gh-aw/agent/github-project-item.json`

If both `sync-from-jira` and `sync-to-jira` are currently present, do not synchronize either direction. Add one comment explaining that the commands conflict and remove both command labels.

## `sync-from-jira`

Jira wins for mapped work-item fields. GitHub remains the developer workspace.

Use `update_issue` exactly once and set `issue_number` to the triggering GitHub issue number:

1. Set the title to `[<Jira key>] <Jira summary>`.
2. Set `operation` to `replace-island`.
3. Set the body to a concise `## Jira source` section containing the Jira key and link, issue type, status, priority, project, assignee, parent, dependencies, updated timestamp, and a note that Jira owns the section.

Only replace this workflow's managed island. Preserve GitHub-owned developer notes, implementation details, checklists, links, pull requests, and discussion.

Then:

- Add `jira-synced`.
- Set GitHub issue field `Priority` from Jira priority when available.
- Set `Effort` from Jira Story Pts: 1-2 → Low, 3-5 → Medium, 8 or more → High.
- Update Project 26 with available Jira values:
  - Jira Status → `Status`: Backlog/Open/To Do → Todo; In Progress/Selected for Development → In progress; Done/Closed/Resolved → Done
  - Jira PI → `PI`
  - Jira Sprint title → `Iteration`
  - Jira Source → `Source`
  - Jira Team → `Team`
  - Jira Requested by → `Requested by`
  - Jira Product category → `Product category`
- Remove `sync-from-jira` only after requesting all GitHub updates.

Do not call `update_jira`.

## `sync-to-jira`

GitHub wins because a developer explicitly selected this direction.

Call `update_jira` exactly once with the triggering issue number, Jira key, and only current values read from the GitHub files:

- GitHub title without `[<Jira key>]` → Jira summary
- Project Status → Jira status
- Priority issue field → Jira priority
- Effort issue field → Jira Story Pts: Low → 2, Medium → 5, High → 8
- Project PI → Jira PI
- Project Iteration title → Jira Sprint
- Project Source → Jira Source
- Project Team → Jira Team
- Project Requested by → Jira Requested by
- Project Product category → Jira Product category

Do not pass `Not mapped`, placeholders, inferred values, IDs, descriptions, or developer notes. Do not remove `sync-to-jira`; the trusted Jira update job removes it only after a successful Jira API update. Do not call GitHub update tools in this direction.

Call `noop` with a short reason for an unsupported command or invalid source data.
