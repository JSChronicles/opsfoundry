# Dependabot Auto-Approval Setup

This guide covers two ways to auto-approve Dependabot pull requests for patch and minor version updates.

Use the single-contributor setup when you fully control the repository and workflow files. Use the multi-contributor setup when other people may contribute to the repository, or when you want a tighter permission boundary around automated approvals.

## High-Level Overview

For the single-contributor setup:
- Enable Allow auto-merge
- Enable Allow GitHub Actions to create and approve pull requests
- Add the workflow


For the multi-contributor GitHub App setup:
- Install the GitHub App on the repository
- Enable Allow auto-merge
- Grant the Dependabot secrets to the repository
- Update CODEOWNERS file
- Add the workflow


## Personal Repository, Single Contributor

Use this setup when you are the only person who can change repository workflows and settings.

This setup uses the built-in `GITHUB_TOKEN`. It is the simplest option and does not require a GitHub App or Dependabot secrets.

### Repository Settings

In the repository, enable auto-merge:

1. Open the repository. > Settings > General > Find Pull Requests
  1.  Check Allow auto-merge

Then allow GitHub Actions to approve pull requests:

1. Open the repository > Settings > Actions > General > Workflow permissions
  1. Check Allow GitHub Actions to create and approve pull requests.

### CODEOWNERS
Add the specific file path to the CODEOWNERS file, with the allowed approver or team as owner. This is to ensure that the dependabot

```yaml
# Protect the workflow that can approve Dependabot PRs with GITHUB_TOKEN.
/.github/workflows/dependabot-auto-approve.yaml @username

# or a team setup
/.github/workflows/dependabot-auto-approve.yaml @your-org/repo-admins
```

### Workflow

Add `dependabot-auto-approve.yaml` to `.github/workflows/`


This workflow can use `secrets.GITHUB_TOKEN` for both approval and auto-merge.

```yaml
name: Dependabot auto-approve

on:
  pull_request:

permissions:
  contents: write
  pull-requests: write

jobs:
  dependabot:
    runs-on: ubuntu-latest

    if: github.event.pull_request.user.login == 'dependabot[bot]'

    env:
      PR_URL: ${{ github.event.pull_request.html_url }}
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    steps:
      - name: Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@25dd0e34f4fe68f24cc83900b1fe3fe149efef98 # v3.1.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Approve patch and minor updates
        if: |
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        run: gh pr review --approve "$PR_URL"

      - name: Enable auto-merge for patch and minor updates
        if: |
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        run: gh pr merge --auto --squash "$PR_URL"
```


## Personal Repository, Multiple Contributors

Use this setup when other contributors may be involved, or when you want automated PR approval handled by a dedicated permission-scoped identity.

Enabling **Allow GitHub Actions to create and approve pull requests** allows workflows in the repository to approve pull requests with `GITHUB_TOKEN`. A GitHub App keeps that approval ability tied to one specific app installation and token.

### Create a GitHub App

Create a GitHub App from your account or organization settings:

1. Open your account or organization settings.
1. Go to **Developer settings** > **GitHub Apps** > Select **New GitHub App**.
1. Use these suggested values:
    1. Name: `dependabot-auto-approval-bot`
    1. Description: `Auto-approves Dependabot patch and minor version updates.`
    1. Homepage URL: `https://github.com/xxxxx`
    1. Webhook: `Disabled`
1. Generate a private key.
1. Save the full private key contents for the Dependabot secret setup.

### GitHub App Permissions

Set these repository permissions:

1. Contents: `Read and write`
1. Pull requests: `Read and write`
1. Metadata: `Read-only`

### Install the GitHub App

Install the app on either all repositories or selected repositories.

!!! note
    Selected repositories is preferred when only some repositories should use this automation.

### Repository Settings

For each repository using the app, enable auto-merge:

1. Open the repository. > Settings > General > Find Pull Requests
  1.  Check Allow auto-merge


### Dependabot Secrets

Add these Dependabot secrets at the repository level or organization level:

1. Open the repository or organization settings.
1. Go to **Secrets and variables**.
  1. Go to **Dependabot**.
    1. Select **New repository secret** or **New organization secret**.
      1. Add `DEPENDABOT_APP_CLIENT_ID` and `DEPENDABOT_APP_PRIVATE_KEY`.

Use the GitHub App client ID and the full private key contents.

If these are organization-level Dependabot secrets, grant access to each repository that should use this automation.

### Workflow

Add `dependabot-auto-approve-multi-contributors.yaml` to `.github/workflows/`


This workflow creates a GitHub App installation token and uses that token for pull request approval and auto-merge.

```yaml
name: Dependabot auto-approve

on:
  pull_request:

permissions:
  contents: read
  pull-requests: read

jobs:
  dependabot:
    runs-on: ubuntu-latest

    if: github.event.pull_request.user.login == 'dependabot[bot]'

    env:
      PR_URL: ${{ github.event.pull_request.html_url }}

    steps:
      - name: Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@25dd0e34f4fe68f24cc83900b1fe3fe149efef98 # v3.1.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Create GitHub App token
        id: app-token
        if: |
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        uses: actions/create-github-app-token@1b10c78c7865c340bc4f6099eb2f838309f1e8c3 # v3.1.1
        with:
          client-id: ${{ secrets.DEPENDABOT_APP_CLIENT_ID }}
          private-key: ${{ secrets.DEPENDABOT_APP_PRIVATE_KEY }}
          permission-pull-requests: write
          permission-contents: write

      - name: Approve patch and minor updates
        if: |
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        run: gh pr review --approve "$PR_URL"
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}

      - name: Enable auto-merge for patch and minor updates
        if: |
          steps.metadata.outputs.update-type == 'version-update:semver-patch' ||
          steps.metadata.outputs.update-type == 'version-update:semver-minor'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
```
