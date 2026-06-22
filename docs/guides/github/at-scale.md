# GitHub at Scale

A few repositories are easy to manage by hand. As the count grows, setup and
maintenance start to take time.

One repository has the right branch protection. Another has an older CI
workflow. Another is missing `SECURITY.md`. Labels mean different things from
repo to repo. New projects start from whatever the last team copied.

This guide is about keeping those things consistent without making every
repository identical. Start with shared defaults, use templates where they help,
and keep exceptions intentional.

## Start With Organization Standards

Before adding more automation, write down the defaults the organization expects
new repositories to follow.

Let's start with the basics:

- repository naming conventions
- required files
- README structure and badges
- default branch name
- CODEOWNERS expectations
- required checks
- release expectations
- issue and pull request labels
- security contact and disclosure process

This does not need to be a large policy document. A short checklist is usually
enough to make repository setup more consistent and easier to review.

## Use Organization-Level Defaults

GitHub supports a special `.github` repository for organization-wide defaults.
This is a good place for files that should apply to most repositories in the
organization.

Common defaults include:

- `profile/README.md`
- `CONTRIBUTING.md`
- `SECURITY.md`
- `SUPPORT.md`
- `CODE_OF_CONDUCT.md`
- default issue templates
- default pull request template

These files give new and existing repositories a shared baseline. If a
repository adds its own version of one of these files, GitHub uses the
repository-local file instead of the organization default.

## Create Repository Templates

Repository templates give teams a better starting point than copying an older
project. Use them for repo types that come up often.

A few good template candidates can include:

- Python packages and services:
  [foundry-python-template](https://github.com/JSChronicles/foundry-python-template)
- Terraform infrastructure modules:
  [foundry-terraform-template](https://github.com/JSChronicles/foundry-terraform-template)
- AWS operational task repositories:
  [foundry-anvil-template](https://github.com/JSChronicles/foundry-anvil-template)
- docs-only repositories
- GitHub Actions workflow repositories

A good template should include the pieces the team would otherwise copy by hand:
README structure, badges, CI, dependency configuration, pre-commit, license,
CODEOWNERS, and a small docs skeleton when the repo needs one.

Use the Anvil task template when the repository is meant to run repeatable AWS
setup, validation, inventory, cleanup, or reporting tasks across accounts,
regions, or organizations.

## Standardize Workflow Patterns

Every repository should not have to invent its workflows from scratch. Define a
few patterns for the repo types the organization uses most, then reuse those
patterns across repositories.

Common patterns include:

- Python format, lint, and test
- Python package build and publish
- documentation build and publish
- Terraform/OpenTofu format, validate, and plan
- Terraform/OpenTofu security scanning
- container build and publish
- dependency review
- CodeQL where it applies

The goal is not to force every repository to run every workflow. The goal is to
make the common workflows predictable so teams know what checks to expect and
maintainers know where to make improvements.

## Use Reusable Workflows

When several repositories need the same workflow logic, move that logic into a
reusable workflow and call it from each repository with `workflow_call`.

This keeps repository workflows smaller. A project can keep a short workflow
file that passes inputs into the shared workflow, while the shared workflow owns
the actual steps.

Reusable workflows should be versioned. For important workflows, prefer calling
a tag or pinned ref instead of a moving branch so updates can be reviewed before
they affect every repository.

## Distribute Standards Across Organizations

Some GitHub setups have more than one organization. This is common in GitHub
Enterprise, but the same problem can show up anywhere a team manages several
organizations.

In that setup, one organization or repository can hold the approved templates,
workflow patterns, and shared files. Other organizations can either consume
those standards directly or receive their own local copies.

### Replicate Template Repositories

In some setups, each organization should have its own copy of the same template
repositories.

For example:

- `OrgA/foundry-python-template`
- `OrgB/foundry-python-template`
- `OrgC/foundry-python-template`

This is useful when each organization needs the template repositories to exist
locally, but the contents should still come from one source of truth.

Treat one organization as the source. Updates to the source template can then be
mirrored into the target organizations.

Use direct mirroring only when the target template repositories should stay
identical to the source. If a target organization needs local changes, use pull
requests or a separate template instead.

### Share Workflow Standards

Reusable workflows do not always need to be copied into every organization. In
GitHub Enterprise, a workflow in one internal or private repository can be
shared with other organizations in the enterprise when access is configured.
GitHub documents this under
[sharing actions and workflows with your enterprise](https://docs.github.com/en/enterprise-cloud@latest/actions/how-tos/reuse-automations/share-with-your-enterprise)
and
[reuse workflows](https://docs.github.com/en/enterprise-cloud@latest/actions/how-tos/reuse-automations/reuse-workflows).

For example, repositories in `OrgB` can call a reusable workflow owned by
`OrgA`:

```yaml
# OrgA/platform-workflows/.github/workflows/python-ci.yml
name: Python CI

on:
  workflow_call:
    inputs:
      python-version:
        required: false
        type: string
        default: "3.14"

jobs:
  pytest:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2

      - name: Install uv
        uses: astral-sh/setup-uv@08807647e7069bb48b6ef5acd8ec9567f424441b # v8.1.0
        with:
          python-version: ${{ inputs.python-version }}
          enable-cache: true
          cache-python: true

      - name: Install the project
        run: uv sync --locked --all-groups

      - name: Run tests with coverage
        run: uv run pytest --cov
```

```yaml
# OrgB/service-api/.github/workflows/ci.yml
name: CI

on:
  pull_request:

jobs:
  ci:
    uses: OrgA/platform-workflows/.github/workflows/python-ci.yml@v1
    with:
      python-version: "3.14"
```

This works well when one organization owns the workflow implementation and
other organizations should consume it. The source repository still needs its
Actions access policy configured so internal or private repositories in the
enterprise are allowed to use the workflow.

Avoid using a minimal `python:*-slim` job container for this shared workflow
unless the image also includes the tools required by the actions themselves.
GitHub runs job steps inside the configured job container, and both
`actions/checkout` and `astral-sh/setup-uv` are JavaScript actions that
currently use Node 24. A plain Python slim image is likely to be missing that
runtime, and may also be missing Git.

### Distribute Shared Files

Some files may still need to live inside each organization or repository.

Useful things to distribute include:

- default issue templates
- default pull request templates
- `SECURITY.md`
- `SUPPORT.md`
- Dependabot defaults
- label sets
- README structure
- CODEOWNERS baselines

Be careful with files that contain local ownership, security contacts,
environment names, or release steps. Those may need per-organization or
per-repository versions.

## Manage Repository Settings Consistently

Files are only part of the setup. Repository settings drift too.

The settings that usually need attention are:

- branch protection and rulesets
- required checks
- squash, rebase, and merge commit settings
- default branch name
- environments
- required reviewers
- vulnerability alerts
- secret scanning
- GitHub Actions permissions
- repository labels

These settings are easy to miss when repositories are created by hand. Keep a
checklist for new repositories, and review older repositories when the standard
changes.

For larger setups, GitHub rulesets and IaC-managed GitHub settings can
help keep repository behavior consistent. Foundry can also help with this later
when the goal is to manage GitHub organizations and repositories as code.

### Standardize Labels and Issue Triage

Labels live on repositories, which means they can drift just like workflows and
settings. If every repository creates its own labels, cross-repo triage gets
messy fast.

Keep labels boring and predictable.

Common label groups include:

- type: `bug`, `feature`, `docs`, `security`
- priority: `p0`, `p1`, `p2`
- status: `blocked`, `needs-info`, `ready`
- area: `ci`, `docs`, `infra`, `deps`

Consistent repository labels make it easier to search across projects, build
reports, and understand what needs attention. Avoid creating a new label every
time one repository has a slightly different workflow.

## Define Security and Compliance Baselines

Some security controls are files, some are repository settings, and some are
workflows. It is still useful to list them together so teams know what the
baseline is.

At minimum, decide what the organization expects for:

- `SECURITY.md`
- private vulnerability reporting
- CodeQL where it applies
- dependency review
- Dependabot
- secret scanning
- branch protection or ruleset requirements
- required reviews
- least-privilege GitHub Actions permissions
- OIDC instead of long-lived cloud secrets

Some repositories will need more controls than others. The baseline should cover
the default expectation, and higher-risk repositories can add stronger
requirements.

## Keep Dependencies Manageable

Dependency updates become noisy when every repository handles them differently.
Set a few expectations so updates are easier to review and less likely to pile
up.

Useful defaults include:

- Dependabot grouped updates
- SHA-pinned GitHub Actions
- scheduled dependency windows
- Dependabot auto-approval, only for low-risk updates
- filtered Dependabot auto-update rules by update type, ecosystem, or dependency prefix
- dependency review for pull requests
- lockfile discipline
- clear ownership for dependency failures

The goal is not to merge every update automatically. It is to keep
dependency work regular and owned by someone or a group.

For more detail, see [Dependabot Best Practices](../dependabot-best-practices.md)
and [Dependabot Auto-Approval](../dependabot-auto-approval.md).

The standard Dependabot auto-approval workflows can handle repositories with multiple ecosystems. Use the filtered Dependabot auto-update workflow when the organization wants an additional policy layer, such as allowing only selected ecosystems, patch and minor updates, or dependency names that match approved prefixes. Pair that with grouped `dependabot.yml` entries per ecosystem so automation stays predictable while still reducing dependency maintenance noise.

## Use Anvil Task Repositories for AWS Operations

Some repositories are not for application code or infrastructure modules. They exist
to run operational work.

For AWS, an Anvil task repository is a good fit when a team needs repeatable
setup, validation, inventory, cleanup, or reporting across accounts, regions, or
organizations.

Use [foundry-anvil-template](https://github.com/JSChronicles/foundry-anvil-template)
as the starting point for that kind of repository. The template already includes
many of the recommended setup files and practices, so teams can start from a
working baseline and adjust it for their needs.

This complements the GitHub standards in this guide. The repository still needs
clear ownership and review.

## When Automation Becomes Necessary

Manual setup works for a while but like mentioned it can start to break down when the same changes
need to be made across many repositories or organizations.

Reasons why you will need automation:

- repositories that need the same update/change
- audits require proof of what is configured
- manual setup keeps drifting
- multiple organizations need the same baseline
- workflow updates are copied by hand
- security policies differ when they should not

At this point, you need something that can handle drift and changes and setup in general.

## Where Foundry Can Help

Everything above can be done without Foundry. For a smaller setup, organization
defaults, repository templates, reusable workflows, rulesets, and a few review
habits may be enough. Start with clear standards and GitHub-native defaults. 
Use Foundry when the manual process becomes too large to keep consistent.
 
Foundry is meant to help manage GitHub as code. Organizations, repositories,
and their settings can be described in YAML files, reviewed through pull
requests, and applied consistently. That makes common setup self-serve for
developers and admins while still keeping control over the approved defaults,
security posture, and review process.

The goal is to make the threshold for proper setup low: copy a best-practice
example, adjust the few values that are local to the team, and let automation
handle the GitHub configuration. Admins and security teams get the same benefit
in the other direction: changes have a Git history, reviews are visible, and it
is easier to see what standards are already supplied across organizations and
repositories instead of guessing from repo to repo.

Foundry can manage items such as:

- repositories
- rulesets
- environments
- labels
- repository settings
- security settings
- template distribution and drift checks
