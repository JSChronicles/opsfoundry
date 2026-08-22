# Examples

The [Anvil examples directory](https://github.com/JSChronicles/anvil/tree/main/examples)
contains schema v2 configurations for every built-in provider.

## AWS

- [Simple organization](https://github.com/JSChronicles/anvil/blob/main/examples/01-aws-simple-organization.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/02-aws-multi-target.yaml)
- [Include and exclude](https://github.com/JSChronicles/anvil/blob/main/examples/03-aws-include-exclude.yaml)
- [Advanced multi-region audit](https://github.com/JSChronicles/anvil/blob/main/examples/04-aws-advanced.yaml)

AWS support is included in the base install.

For dependency result sharing, recovery, and configured-target fan-in, see the
[AWS Config cleanup workflow](https://github.com/JSChronicles/anvil/blob/main/examples/33-aws-config-cleanup-workflow.yaml).

## Azure

- [Simple subscription](https://github.com/JSChronicles/anvil/blob/main/examples/05-azure-simple-subscription.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/06-azure-multi-target.yaml)
- [Tenant include and exclude](https://github.com/JSChronicles/anvil/blob/main/examples/07-azure-include-exclude.yaml)
- [Advanced resource-group inventory](https://github.com/JSChronicles/anvil/blob/main/examples/08-azure-advanced.yaml)

Install the Azure SDK dependencies with `pip install "anvil[azure]"`, or use
`uv sync --extra azure` in an Anvil source checkout.

## GCP

- [Simple project](https://github.com/JSChronicles/anvil/blob/main/examples/09-gcp-simple-project.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/10-gcp-multi-target.yaml)
- [Project include selection](https://github.com/JSChronicles/anvil/blob/main/examples/11-gcp-include-exclude.yaml)
- [Advanced project inventory](https://github.com/JSChronicles/anvil/blob/main/examples/12-gcp-advanced.yaml)

Install the GCP SDK dependencies with `pip install "anvil[gcp]"`, or use
`uv sync --extra gcp` in an Anvil source checkout.

## GitHub

- [Simple repository](https://github.com/JSChronicles/anvil/blob/main/examples/13-github-simple-repository.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/14-github-multi-target.yaml)
- [Organization and repository selection](https://github.com/JSChronicles/anvil/blob/main/examples/15-github-include-exclude.yaml)
- [Advanced security audit with GitHub App auth](https://github.com/JSChronicles/anvil/blob/main/examples/16-github-advanced.yaml)

GitHub examples cover token auth, GitHub App auth, organization-scoped code
search, repository security settings, rulesets, branch protection, and security
alert listing.

Install the GitHub client dependency with `pip install "anvil[github]"`, or use
`uv sync --extra github` in an Anvil source checkout.

## Cloudflare

- [Simple account](https://github.com/JSChronicles/anvil/blob/main/examples/17-cloudflare-simple-account.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/18-cloudflare-multi-target.yaml)
- [Account and zone selection](https://github.com/JSChronicles/anvil/blob/main/examples/19-cloudflare-include-exclude.yaml)
- [Advanced DNS inventory](https://github.com/JSChronicles/anvil/blob/main/examples/20-cloudflare-advanced.yaml)

Install the Cloudflare client with `pip install "anvil[cloudflare]"`, or use
`uv sync --extra cloudflare` in an Anvil source checkout.

## Datadog

- [Simple organization](https://github.com/JSChronicles/anvil/blob/main/examples/21-datadog-simple-organization.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/22-datadog-multi-target.yaml)
- [Task-level resource selection](https://github.com/JSChronicles/anvil/blob/main/examples/23-datadog-include-exclude.yaml)
- [Advanced observability audit](https://github.com/JSChronicles/anvil/blob/main/examples/24-datadog-advanced.yaml)

Install the Datadog client with `pip install "anvil[datadog]"`, or use
`uv sync --extra datadog` in an Anvil source checkout.

## GitLab

- [Simple project](https://github.com/JSChronicles/anvil/blob/main/examples/25-gitlab-simple-project.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/26-gitlab-multi-target.yaml)
- [Group and project selection](https://github.com/JSChronicles/anvil/blob/main/examples/27-gitlab-include-exclude.yaml)
- [Advanced security audit](https://github.com/JSChronicles/anvil/blob/main/examples/28-gitlab-advanced.yaml)

Install the GitLab client with `pip install "anvil[gitlab]"`, or use
`uv sync --extra gitlab` in an Anvil source checkout.

## PagerDuty

- [Simple account](https://github.com/JSChronicles/anvil/blob/main/examples/29-pagerduty-simple-account.yaml)
- [Multiple targets](https://github.com/JSChronicles/anvil/blob/main/examples/30-pagerduty-multi-target.yaml)
- [Task-level resource selection](https://github.com/JSChronicles/anvil/blob/main/examples/31-pagerduty-include-exclude.yaml)
- [Advanced operations audit](https://github.com/JSChronicles/anvil/blob/main/examples/32-pagerduty-advanced.yaml)

Install the PagerDuty client with `pip install "anvil[pagerduty]"`, or use
`uv sync --extra pagerduty` in an Anvil source checkout.

## GitHub Actions

For AWS OIDC workflows that execute Anvil and upload generated JSON results as
workflow artifacts, see the
[GitHub Actions examples](https://github.com/JSChronicles/anvil/blob/main/examples/github-actions/README.md).

## Result Examples

The [result examples](https://github.com/JSChronicles/anvil/tree/main/examples/Results)
show structured task return values and `ActionRecorder` usage.

## Repository Template

Create a dedicated task repository with the
[foundry-anvil-template](https://github.com/JSChronicles/foundry-anvil-template).
It provides a project layout for custom tasks and processors, YAML examples,
validation, and CI outside the main Anvil repository.

## Standalone AWS Multi-Account Template

If you need a small AWS Organizations script instead of the Anvil framework,
start with the
[standalone multi-account template](templates/aws_multi_account_template.py).

It includes active-account discovery, include/exclude filters, parallel account
execution, multiple regions, member-account role assumption, dry-run handling,
and JSON output. Replace `account_task()` with the work for your script and
adapt the example arguments as needed.
