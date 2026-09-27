# Soamee GitHub Actions

Reusable workflows and composite actions shared across all Soamee repos.

## Private registry

`@soamee/*` packages are published privately to GitHub Packages. Every repo that
installs them commits this `.npmrc`:

```
@soamee:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NPM_TOKEN}
```

`NPM_TOKEN` is an organization secret holding a classic personal access token with
`read:packages`. Yarn 1 expands `${NPM_TOKEN}` whenever it loads config, so every
step that runs `yarn` fails without it, including `actions/setup-node` with
`cache: yarn`. Set it in the job's `env`, not per step, as the workflows here do.
A Docker build in a caller's own CI receives it as a BuildKit secret
(`--secret id=npm_token,env=NPM_TOKEN`) so it never lands in an image layer; a
Dokku deploy builds on the server, so the token must be configured on the Dokku
app. Publishing uses the workflow's own `GITHUB_TOKEN`. To install locally, export `NPM_TOKEN` with your own classic token
(`read:packages`).

## Pinning

The examples below use `@main` for readability. In a repo, pin the full commit SHA
and keep `# main` as a trailing comment:

```yaml
uses: soamee/github-actions/.github/workflows/version-bump.yml@<sha> # main
```

Callers pass `secrets: inherit`, and `version-bump` also gets `packages: write`, so a
floating `@main` gives whoever can push here the secrets of every caller. Moving to a newer
version means bumping the SHA in each caller.

## Reusable Workflows

### Update Soamee Packages

Checks GitHub Packages for newer `@soamee/*` releases, updates package.json, and creates a PR.

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

### Release: Promote + Version Bump & Publish

`promote.yml` merges the development branch into the release branch, and
`version-bump.yml` bumps the version on that release branch, builds, tests,
publishes to GitHub Packages, tags and creates the GitHub Release. Chain both so a single
manual run promotes `develop` to `main` and publishes.

```yaml
# .github/workflows/version-bump.yml
name: "Version Bump & Publish"

on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version bump type"
        required: true
        type: choice
        options:
          - patch
          - minor
          - major
      source-branch:
        description: "Branch promoted into main before bumping (set to 'main' to skip the promotion)"
        required: false
        type: string
        default: develop

jobs:
  promote:
    if: inputs.source-branch != 'main'
    uses: soamee/github-actions/.github/workflows/promote.yml@main
    with:
      source-branch: ${{ inputs.source-branch }}
      target-branch: main
      mode: merge
    secrets: inherit

  bump-and-publish:
    needs: promote
    if: ${{ always() && (needs.promote.result == 'success' || needs.promote.result == 'skipped') }}
    permissions:
      contents: write
      packages: write
    uses: soamee/github-actions/.github/workflows/version-bump.yml@main
    with:
      version: ${{ inputs.version }}
      publish-branch: main
    secrets: inherit
```

**`promote.yml` inputs:**
| Input | Default | Description |
|-------|---------|-------------|
| `source-branch` | `develop` | Branch to promote from |
| `target-branch` | `main` | Branch to promote into |
| `mode` | `merge` | `merge` pushes directly, `pr` opens a promotion PR |

**`version-bump.yml` inputs:**
| Input | Default | Description |
|-------|---------|-------------|
| `version` | *required* | `patch`, `minor` or `major` |
| `node-version` | `22` | Node.js version |
| `package-manager` | `npm` | `npm` or `yarn` |
| `build-command` | `npm run build` | Build command |
| `test-command` | `npm test -- --runInBand` | Empty to skip tests |
| `publish-branch` | `main` | Branch the release commit and tag land on |

Both workflows push with `DEPENDENCY_UPDATE_TOKEN` when available (needed when
the release branch is protected) and fall back to `GITHUB_TOKEN`.

## Composite Actions

### setup-node-cache

Node.js + Yarn install with node_modules caching. The job needs `NPM_TOKEN` in its
`env` if the repo installs `@soamee/*` packages.

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

Runs linter with --fix, commits and pushes changes. Like `setup-node-cache`, the job
needs `NPM_TOKEN` in its `env` if the repo installs `@soamee/*` packages.

```yaml
- uses: soamee/github-actions/actions/lint-autofix@main
  with:
    lint-command: "yarn lint --fix"
```
