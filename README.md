# Agentic work item sync demo

This public demo shows a label-triggered, AI-assisted synchronization pattern where **Jira is the system of record** and **GitHub is the developer workspace**.

## Demo flow

1. A GitHub issue references a Jira key such as `SHIP-123`.
2. Apply the `preview-jira-refresh` label.
3. The label starts the `jira-source-preview` agentic workflow and is automatically removed so it can be used again.
4. Copilot reads the Jira-shaped source record from `.demo/jira/SHIP-123.json`.
5. Copilot compares Jira-owned fields with the GitHub issue and posts a refresh preview.

The preview separates:

- **Jira-owned fields:** title, work item type, status, priority, project, team, assignee, parent, and dependencies
- **GitHub-owned fields:** developer notes, implementation details, local checklists, links, and discussion

The workflow does not write to Jira and does not overwrite the GitHub issue. It demonstrates the source-of-truth direction and the field ownership model before an integration adapter is added.

## Authentication, model, and billing

The workflow uses:

```yaml
engine: copilot
model: auto
permissions:
  contents: read
  issues: read
  copilot-requests: write
```

`copilot-requests: write` lets gh-aw use `${{ github.token }}` for Copilot inference. No personal access token or `COPILOT_GITHUB_TOKEN` secret is required. The organization must allow Copilot CLI requests billed to the organization.

## Try it

Open the [sample issue](../../issues/2), then apply the `preview-jira-refresh` label. The workflow reads `.demo/jira/SHIP-123.json` and posts the proposed Jira-to-GitHub changes.

To create another example:

1. Add a Jira-shaped JSON fixture under `.demo/jira/<KEY>.json`.
2. Create a GitHub issue containing `Jira key: <KEY>`.
3. Add developer-owned notes or implementation fields.
4. Apply `preview-jira-refresh`.

## Production extension

Replace the fixture read with a deterministic Jira Cloud adapter:

1. Fetch `/rest/api/3/issue/{key}` in a trusted workflow step or MCP server.
2. Store `JIRA_BASE_URL`, `JIRA_EMAIL`, and `JIRA_API_TOKEN` as GitHub Actions secrets.
3. Pass only the normalized Jira response to the agent.
4. Use constrained safe outputs to update Jira-owned GitHub fields.
5. Preserve GitHub-owned sections and comments.
6. Add idempotency, conflict detection, retries, and audit logging.

The same source-of-truth pattern can target Digital.ai Agility or Azure DevOps by replacing the read adapter and field mappings.

## Local authoring

```bash
gh extension install github/gh-aw
gh aw compile jira-source-preview
gh aw validate jira-source-preview
```

The repository is initialized for agentic authoring, so the `agentic-workflows` custom agent can create, update, and debug workflows from GitHub Copilot.
