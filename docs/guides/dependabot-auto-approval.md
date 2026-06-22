# Dependabot Auto-Approval Setup

This guide covers two ways to auto-approve Dependabot pull requests for patch and minor version updates.

Use the single-contributor setup when you fully control the repository and workflow files. Use the multi-contributor setup when other people may contribute to the repository, or when you want a tighter permission boundary around automated approvals.

Use the filtered auto-update setup when a repository needs more control over which dependency ecosystems, dependency name prefixes, and semantic version update types are allowed to receive automatic approval and auto-merge.

## High-Level Overview

For the single-contributor setup:
- Enable Allow auto-merge
- Enable Allow GitHub Actions to create and approve pull requests
- Add the workflow


For the multi-contributor GitHub App setup:
- Install the GitHub App on the repository
- Enable Allow auto-merge
- Grant the Dependabot secrets to the repository
- Add the workflow

For the filtered auto-update setup:
- Follow the setup as needed to match single-contributor or multi-contributor
- Decide which update types are allowed, such as patch and minor updates.
- Decide whether to include or exclude dependency name prefixes.
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

Add `dependabot-auto-approve.yaml` to `.github/workflows/`


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

## Filtered Auto-Update Workflow

Use this workflow when auto-approval should be limited by update type and dependency name. It is useful for repositories that want broad Dependabot coverage but only want low-risk ecosystems or dependency groups to be auto-approved and auto-merged.

The default example allows only patch and minor updates. `INCLUDED_PREFIXES_JSON` and `EXCLUDED_PREFIXES_JSON` can be used to allow or block dependency names by prefix.

```yaml
name: Dependabot auto-update

on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened
      - ready_for_review

permissions:
  contents: write
  pull-requests: write

jobs:
  dependabot-auto-update:
    if: ${{ github.event.pull_request.user.login == 'dependabot[bot]' }}
    runs-on: ubuntu-latest
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      PR_URL: ${{ github.event.pull_request.html_url }}
      UPDATE_TYPES_JSON: '["semver-patch","semver-minor"]'
      INCLUDED_PREFIXES_JSON: '[]'
      EXCLUDED_PREFIXES_JSON: '[]'

    steps:
      - name: Fetch Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@25dd0e34f4fe68f24cc83900b1fe3fe149efef98 # v3.1.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Check eligibility
        id: eligibility
        env:
          UPDATE_TYPE: ${{ steps.metadata.outputs.update-type }}
          DEPENDENCY_NAMES: ${{ steps.metadata.outputs.dependency-names }}
        uses: actions/github-script@3a2844b7e9c422d3c10d287c895573f7108da1b3 # v9.0.0
        with:
          script: |
            const updateType = process.env.UPDATE_TYPE.replace(/^version-update:/, "");
            const dependencyNames = process.env.DEPENDENCY_NAMES
              .split(/[\n,]+/)
              .map((name) => name.trim())
              .filter(Boolean);
            const updateTypes = new Set(JSON.parse(process.env.UPDATE_TYPES_JSON));
            const includedPrefixes = JSON.parse(process.env.INCLUDED_PREFIXES_JSON);
            const excludedPrefixes = JSON.parse(process.env.EXCLUDED_PREFIXES_JSON);
            const included =
              includedPrefixes.length === 0 ||
              dependencyNames.every((name) =>
                includedPrefixes.some((prefix) => name.startsWith(prefix))
              );
            const excluded = dependencyNames.some((name) =>
              excludedPrefixes.some((prefix) => name.startsWith(prefix))
            );

            core.setOutput("eligible", updateTypes.has(updateType) && included && !excluded);

      - name: Approve pull request
        if: ${{ steps.eligibility.outputs.eligible == 'true' }}
        run: gh pr review "$PR_URL" --approve --body "Foundry-managed Dependabot approval"

      - name: Enable auto-merge
        if: ${{ steps.eligibility.outputs.eligible == 'true' }}
        run: gh pr merge "$PR_URL" --auto --squash
```

Keep the workflow protected with CODEOWNERS and branch protection or rulesets. Anyone who can change this workflow can change what gets approved and merged automatically.

### CODEOWNERS

Add the auto-approval workflow path to the CODEOWNERS file, with the repository admin, maintainer, or trusted maintainer team as owner. Pair this with branch protection or rulesets that require code owner review for workflow changes.

This keeps contributors from changing the workflow that can mint the GitHub App token and approve Dependabot pull requests.

```yaml
# Protect the workflow that can approve Dependabot PRs with the GitHub App token.
/.github/workflows/dependabot-auto-approve.yaml @username

# or a team setup
/.github/workflows/dependabot-auto-approve.yaml @your-org/repo-admins
```