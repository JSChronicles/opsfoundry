# Provider Reference

Providers own authentication, location discovery, target resolution, and the
session passed to tasks. Use `anvil list --providers` to inspect installed
providers and `anvil validate --providers` to validate their contracts.

Install only the SDK extras required by your workflows:

```console
pip install anvil
pip install "anvil[azure]"
pip install "anvil[cloudflare]"
pip install "anvil[datadog]"
pip install "anvil[gcp]"
pip install "anvil[github]"
pip install "anvil[gitlab]"
pip install "anvil[pagerduty]"
```

AWS support is included in the base installation.

## At a Glance

| Provider | Modes | Default location | Target filtering |
| --- | --- | --- | --- |
| AWS | `organization`, `accounts` | `us-east-1` | Discovery include/exclude or explicit account IDs |
| Azure | `tenant`, `subscriptions` | `eastus` | Discovery include/exclude or explicit subscription IDs |
| Cloudflare | `accounts`, `zones` | `global` | Discovery include/exclude or explicit IDs |
| Datadog | `organization` | `global` | None; one organization per target |
| GCP | `projects`; `organization` reserved | `us-central1` | Project discovery or explicit project IDs |
| GitHub | `organizations`, `repositories` | `global` | Explicit include required |
| GitLab | `groups`, `projects` | `global` | Discovery include/exclude or explicit IDs/paths |
| PagerDuty | `account` | `global` | None; one account per target |

See [selectors and regions](selectors-and-regions.md) for complete filtering,
special keyword, `all`, and glob rules.

The [built-in components](built-in-components.md) page lists the stock tasks
for every provider, their target assumptions, and their important metadata
inputs.

## Authentication Check Depth

`anvil validate --auth --config-file targets.yaml` always validates the target
shape and configured credential source, but not every provider performs a live
request at this phase:

| Provider | Authentication validation |
| --- | --- |
| AWS | Live STS identity check |
| Azure | Live Azure Resource Manager token acquisition |
| Cloudflare | Credential resolution and client construction; permissions deferred |
| Datadog | Live API/application-key validation |
| GCP | Deferred until runtime session construction |
| GitHub | Token or App setting resolution; no API request |
| GitLab | Live authenticated API request |
| PagerDuty | Token resolution and client construction; no API request |

Discovery and task execution can still reveal resource-level permission gaps
after an authentication check succeeds.

## AWS

Use `organization` to discover active organization accounts. Use `accounts`
for explicit account IDs. `profile` selects a boto3 profile, and `role_name`
enables role assumption into selected accounts.

```yaml
provider:
  name: aws
  mode: organization
  options:
    profile: security-audit
    role_name: OrganizationAccountAccessRole
exclude:
  - payer
regions:
  - us-*
# ...
```

In `accounts` mode, `include` is required. Without `role_name`, exactly one
account must be selected and the current credentials must resolve directly to
that account. AWS tasks receive a boto3-compatible session scoped to the
current account and Region.

## Azure

Use `tenant` to discover subscriptions or `subscriptions` with an explicit
`include` list. Omit auth options to use `DefaultAzureCredential`.

```yaml
provider:
  name: azure
  mode: tenant
  options:
    tenant_id: 11111111-2222-3333-4444-555555555555
include:
  - aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
regions:
  - eastus
# ...
```

For client-secret auth, provide `tenant_id`, `client_id`, and `client_secret`
together. Without `client_secret`, `client_id` can select a managed identity.
Azure tasks receive the credential, subscription ID, and location context.

## Cloudflare

Use `accounts` for account-member or zone inventory and `zones` for DNS work.
Targets use only `global`.

```yaml
provider:
  name: cloudflare
  mode: zones
  options:
    api_token_env: CLOUDFLARE_API_TOKEN
    account_id: '11111111111111111111111111111111'
include:
  - aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
regions:
  - global
# ...
```

`account_id` is valid only in `zones` mode. Authenticate with `api_token_env`
or the paired `api_key_env` and `api_email_env`. `base_url` supports alternate
API endpoints. Shared [provider profiles](provider-profiles.md) are supported.

## Datadog

Each target represents one key-bound Datadog organization. `include` and
`exclude` are not supported, and the location is always `global`.

```yaml
provider:
  name: datadog
  mode: organization
  options:
    site: datadoghq.eu
    api_key_env: DD_API_KEY
    app_key_env: DD_APP_KEY
regions:
  - global
# ...
```

Use `site` as a hostname, not a URL. Configure multiple top-level targets and
profiles for multiple organizations or sites.

## GCP

Use `projects` with explicit project IDs or omit `include` to discover
accessible projects. `organization` and `organization_id` are reserved, but
organization discovery is not implemented.

```yaml
provider:
  name: gcp
  mode: projects
  options:
    quota_project_id: anvil-billing-project
include:
  - anvil-production
regions:
  - us-*
# ...
```

Omit `credentials_path` to use application-default credentials. When supplied,
it selects a credentials file. GCP resolves Compute regions whose status is
`UP` and passes project, quota-project, credential, and region context to tasks.

## GitHub

Both modes require `include`. Organization values are owner logins; repository
values use `owner/repository`. `exclude` is not supported, and the location is
always `global`.

```yaml
provider:
  name: github
  mode: repositories
  options:
    token_env: GITHUB_TOKEN
include:
  - octo-org/platform-api
regions:
  - global
# ...
```

Authenticate with a token or a GitHub App using `app_id` plus exactly one of
`private_key_env` or `private_key_path`. `api_url` and `api_version` support
GitHub Enterprise and explicit API versions. Shared profiles are supported.

## GitLab

Use `groups` or `projects`. Omit selection to discover visible resources, use
`include` for exact numeric IDs or paths, or use `exclude` with discovery.

```yaml
provider:
  name: gitlab
  mode: projects
  options:
    token_env: GITLAB_TOKEN
    url: https://gitlab.example.com
include:
  - platform/application-api
regions:
  - global
# ...
```

`token_env` is required directly or through a profile. `auth_type` accepts
`private` or `oauth`, and `ca_cert_path` supports private certificate
authorities. GitLab targets use only `global`.

## PagerDuty

Each target represents one PagerDuty account. Target-level `include` and
`exclude` are not supported, and the location is always `global`.

```yaml
provider:
  name: pagerduty
  mode: account
  options:
    token_env: PAGERDUTY_API_TOKEN
    subdomain: acme
regions:
  - global
# ...
```

`auth_type` accepts `token` or `bearer`. Use `api_url` for regional endpoints
and `from_email` when an API operation requires a requester identity. Configure
one top-level target per account.

## Validate Before Running

```console
anvil validate --providers
anvil validate --config-file targets.yaml
anvil validate --auth --config-file targets.yaml
```

Provider contract and configuration validation are offline. Authentication
validation calls the selected provider's access check but does not run tasks.
