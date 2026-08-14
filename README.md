# Agentic work item sync demo

This public demo shows how a GitHub label can trigger an AI-assisted work item synchronization workflow. It uses [GitHub Agentic Workflows](https://github.github.com/gh-aw/) and the built-in `${{ github.token }}` for Copilot inference.

## Demo flow

1. Open a GitHub issue that represents a work item.
2. Apply the `sync-to-agility` label.
3. The label starts the `work-item-sync` agentic workflow and is automatically removed so it can be used again.
4. Copilot analyzes the issue, produces a normalized Digital.ai Agility-style payload, and posts a sync preview as an issue comment.
5. The preview includes field mappings, hierarchy and dependency notes, validation warnings, and the API operation an integration adapter would perform.

The workflow intentionally stops before calling Digital.ai Agility. This keeps the demo safe and makes the integration boundary clear. A production implementation would replace the preview with an approved MCP server or custom safe-output job that owns the Agility API call and credential.

## Authentication and billing

The workflow declares:

```yaml
permissions:
  contents: read
  issues: read
  copilot-requests: write
```

`copilot-requests: write` lets gh-aw use `${{ github.token }}` for Copilot inference. No personal access token or `COPILOT_GITHUB_TOKEN` secret is required. The organization must allow Copilot CLI requests billed to the organization.

## Try it

Use the sample issue, or create an issue with fields such as:

```markdown
## Description
Add reusable shipment notification preferences.

## Acceptance criteria
- Users can select email or SMS.
- Preferences persist across sessions.

## External mapping
- Project: Customer Experience
- Team: Delivery Notifications
- Type: Story
- Parent: GH-100
```

Apply the `sync-to-agility` label and watch the Actions run.

## Production extension

The next step is an integration adapter with:

- a Digital.ai Agility MCP server or custom safe-output job
- a repository or organization secret for the Agility token
- deterministic external ID storage for create versus update decisions
- status, team, project, hierarchy, and relationship mappings
- idempotency, retries, conflict handling, and audit logging

The same pattern can target Jira Cloud or Azure DevOps by changing the adapter while keeping the label trigger and normalization prompt.

## Local authoring

```bash
gh extension install github/gh-aw
gh aw compile work-item-sync
gh aw validate work-item-sync
```

The repository is initialized for agentic authoring, so the `agentic-workflows` custom agent can also create, update, and debug workflows from GitHub Copilot.
