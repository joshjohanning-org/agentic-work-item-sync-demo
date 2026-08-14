# Agentic work item sync demo

This public demo shows a label-triggered, AI-assisted synchronization pattern where **Jira is the system of record** and **GitHub is the developer workspace**.

## Demo flow

1. A GitHub issue references a Jira key such as `GHAW-1`.
2. Apply the `preview-jira-refresh` label.
3. The label starts the `jira-source-preview` agentic workflow and is automatically removed so it can be used again.
4. A trusted workflow step reads the source issue from Jira Cloud using `secrets.JIRA_TOKEN`.
5. Copilot compares Jira-owned fields with the GitHub issue and posts a refresh preview.

The preview separates:

- **Jira-owned fields:** title, work item type, status, priority, project, team, assignee, parent, and dependencies
- **GitHub-owned fields:** developer notes, implementation details, local checklists, links, and discussion

The workflow reads Jira but does not write to Jira or overwrite the GitHub issue. It demonstrates the source-of-truth direction and the field ownership model before constrained GitHub updates are enabled.

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

Open the [sample issue](../../issues/2), then apply the `preview-jira-refresh` label. The workflow reads the referenced `GHAW-*` issue from Jira Cloud and posts the proposed Jira-to-GitHub changes.

The repository needs:

- `JIRA_TOKEN`: Jira API token or OAuth access token
- `JIRA_EMAIL`: optional Atlassian account email when `JIRA_TOKEN` is an API token that requires Basic authentication

To create another example:

1. Create a GitHub issue containing `Jira key: GHAW-<number>`.
2. Add developer-owned notes or implementation fields.
3. Apply `preview-jira-refresh`.

## Production extension

Extend the deterministic Jira Cloud read adapter:

1. Pass only the normalized Jira response to the agent.
2. Add field ID mappings for team and organization-specific custom fields.
3. Use constrained safe outputs to update Jira-owned GitHub fields.
4. Preserve GitHub-owned sections and comments.
5. Add idempotency, conflict detection, retries, and audit logging.

The same source-of-truth pattern can target Digital.ai Agility or Azure DevOps by replacing the read adapter and field mappings.

## Local authoring

```bash
gh extension install github/gh-aw
gh aw compile jira-source-preview
gh aw validate jira-source-preview
```

The repository is initialized for agentic authoring, so the `agentic-workflows` custom agent can create, update, and debug workflows from GitHub Copilot.
