# Soamee GitHub Actions

Reusable workflows and composite actions shared across all Soamee repos.

## Reusable Workflows

### Update Soamee Packages

Checks npm for newer `@soamee/*` releases, updates package.json, and creates a PR.

```yaml
# .github/workflows/update-soamee-packages.yml
name: "Update Soamee Packages"

on:
  schedule:
    - cron: "0 6 * * 2,5"
  workflow_dispatch:

jobs:
  update:
    uses: soamee/github-actions/.github/workflows/update-soamee-packages.yml@main
    with:
      base-branch: develop
      node-version: "22"
    secrets: inherit
```

**Inputs:**
| Input | Default | Description |
|-------|---------|-------------|
| `base-branch` | `develop` | Target branch for PRs |
| `node-version` | *required* | Node.js version |
| `reviewers` | Team list | Space-separated GitHub usernames |

## Composite Actions

### setup-node-cache

Node.js + Yarn install with node_modules caching.

```yaml
- uses: soamee/github-actions/actions/setup-node-cache@main
  with:
    node-version: "22"
```

### slack-notify

Single step replacing the 3-step success/cancel/fail pattern.

```yaml
- uses: soamee/github-actions/actions/slack-notify@main
  if: always()
  with:
    status: ${{ job.status }}
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
```

### lint-autofix

Runs linter with --fix, commits and pushes changes.

```yaml
- uses: soamee/github-actions/actions/lint-autofix@main
  with:
    lint-command: "yarn lint --fix"
```
