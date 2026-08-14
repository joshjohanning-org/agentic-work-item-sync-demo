# Jira to GitHub agentic sync demo

This public demo models **Jira as the source of truth** and **GitHub as the developer workspace**.

## Workflows

### Import new Jira issues

`.github/workflows/jira-import-new.md` runs every six hours and supports manual dispatch.

It:

1. Reads recent issues from the Jira `GHAW` project.
2. Searches GitHub for existing issues containing each Jira key.
3. Creates up to three missing GitHub issues.
4. Adds a Jira-owned source section and a GitHub-owned developer notes section.

### Sync from Jira on command

`.github/workflows/jira-sync-on-command.md` runs when someone applies the `sync-from-jira` label.

It:

1. Extracts the `GHAW-*` Jira key from the GitHub issue.
2. Reads the current Jira issue.
3. Updates the GitHub title and the workflow-managed Jira source section.
4. Preserves developer notes, implementation details, checklists, links, and discussion.
5. Adds the `jira-synced` label.

The command label is automatically removed after each run so it can be applied again.

## Field ownership

| Owner | Fields |
|---|---|
| Jira | Key, summary, type, status, priority, PI, sprint, story points, source, team, requested by, product category, project, assignee, parent, dependencies, labels, timestamps |
| GitHub | Developer notes, implementation details, checklists, links, pull requests, and discussion |

The demo only reads Jira. It never writes changes back to Jira after the source issues have been seeded.

## Authentication, model, and billing

Both agentic workflows use:

```yaml
engine: copilot
model: auto
permissions:
  contents: read
  issues: read
  copilot-requests: write
```

`copilot-requests: write` uses `${{ github.token }}` for Copilot inference and organization billing.

Repository secrets:

- `JIRA_TOKEN`: Jira API token or OAuth access token
- `JIRA_EMAIL`: Atlassian account email when `JIRA_TOKEN` requires Basic authentication

GitHub App configuration for automatic Project writes:

- Repository variable `PROJECT_APP_CLIENT_ID`
- Repository secret `PROJECT_APP_PRIVATE_KEY`
- App repository permission: Issues read/write
- App organization permission: Projects read/write

The Jira network allowlist is restricted to `joshjohanning.atlassian.net`.

`GHAW` is a team-managed Jira project. Jira Cloud does not expose an API for adding global custom fields to a team-managed work type layout. After running **Configure Jira Demo Fields**, add `PI`, `Story Pts`, `Source`, `Requested by`, and `Product category` once through **Project settings → Work types → Task**. Jira already provides Priority, Sprint, and Team.

Project mappings:

| Jira | GitHub |
|---|---|
| Status | Project Status |
| Priority | Priority issue field |
| Story Pts | Effort issue field: 1-2 Low, 3-5 Medium, 8+ High |
| PI | Project PI |
| Sprint | Project Iteration |
| Source | Project Source, representing work origin |
| Team | Project Team |
| Requested by | Project Requested by |
| Product category | Project Product category |

## Demo setup

1. Run **Seed Jira Demo Issues** once to create three `gh-aw-demo` issues in Jira.
2. Run **Import New Jira Issues** to create their GitHub developer issues.
3. Change a source issue in Jira.
4. Apply `sync-from-jira` to the matching GitHub issue.
5. Watch the Jira-owned section refresh while developer content remains unchanged.

## Local authoring

```bash
gh extension install github/gh-aw
gh aw compile jira-import-new jira-sync-on-command
gh aw validate jira-import-new jira-sync-on-command
```

The repository is initialized for agentic authoring, so the `agentic-workflows` custom agent can create, update, and debug workflows from GitHub Copilot.
