# Provider Profiles

Provider profiles keep reusable authentication and endpoint settings outside
workflow YAML. They live in `~/.anvil/config.toml`; set `ANVIL_CONFIG` to use a
different file.

Profiles are namespaced by provider and profile name:

```toml
[providers.cloudflare.security]
api_token_env = "CLOUDFLARE_SECURITY_TOKEN"

[providers.github.work]
token_env = "GITHUB_WORK_TOKEN"
api_url = "https://api.github.com"

[providers.gitlab.default]
token_env = "GITLAB_TOKEN"
url = "https://gitlab.example.com"
```

Profile fields should reference environment-variable names or provider-native
credential locations. Do not store tokens, keys, or client secrets in this
file.

## Named and Default Profiles

Select a named profile in target options:

```yaml
provider:
  name: github
  mode: repositories
  options:
    profile: work
# ...
```

A profile named `default` is selected automatically when no named profile or
inline profile fields are configured:

```yaml
provider:
  name: gitlab
  mode: projects
  options: {}
# Uses providers.gitlab.default.
```

A named profile cannot be mixed with inline profile fields such as
`token_env`, `api_url`, `site`, or `url`. Provider-specific selectors that are
not profile fields remain inline.

## Cloudflare: Shared Auth, Inline Account Selection

Use one authentication profile with different account or zone selections:

```toml
[providers.cloudflare.security]
api_token_env = "CLOUDFLARE_SECURITY_TOKEN"
base_url = "https://api.cloudflare.com/client/v4"
```

```yaml
provider:
  name: cloudflare
  mode: zones
  options:
    profile: security
    account_id: '11111111111111111111111111111111'
include:
  - aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
# ...
```

`account_id` is a target selector, so it can accompany `profile`. Cloudflare
profiles support token authentication or the paired global API key and email
environment-variable names.

## GitHub: Token and App Profiles

Separate human or automation token access from GitHub App access:

```toml
[providers.github.readonly]
token_env = "GITHUB_READONLY_TOKEN"

[providers.github.security_app]
app_id = "12345"
private_key_env = "GITHUB_APP_PRIVATE_KEY"
api_url = "https://api.github.com"
api_version = "2022-11-28"
```

```yaml
provider:
  name: github
  mode: repositories
  options:
    profile: security_app
include:
  - octo-org/platform-api
# ...
```

Use exactly one of `private_key_env` or `private_key_path` for app
authentication.

## GitLab: Multiple Instances and Auth Types

Profiles are useful when workflows span GitLab.com and a self-managed instance:

```toml
[providers.gitlab.default]
token_env = "GITLAB_COM_TOKEN"

[providers.gitlab.corporate]
url = "https://gitlab.example.com"
auth_type = "oauth"
token_env = "GITLAB_CORPORATE_TOKEN"
ca_cert_path = "C:/certificates/corporate-ca.pem"
```

```yaml
provider:
  name: gitlab
  mode: groups
  options:
    profile: corporate
include:
  - platform/security
# ...
```

GitLab supports `private` and `oauth` auth types.

## Datadog: Organizations on Different Sites

Configure one target and profile per key-bound organization:

```toml
[providers.datadog.production]
site = "datadoghq.com"
api_key_env = "DD_PRODUCTION_API_KEY"
app_key_env = "DD_PRODUCTION_APP_KEY"

[providers.datadog.europe]
site = "datadoghq.eu"
api_key_env = "DD_EU_API_KEY"
app_key_env = "DD_EU_APP_KEY"
```

```yaml
provider:
  name: datadog
  mode: organization
  options:
    profile: europe
# ...
```

## PagerDuty: Regional API Endpoints

Use separate profiles for account credentials and endpoints:

```toml
[providers.pagerduty.production]
token_env = "PAGERDUTY_PRODUCTION_TOKEN"
subdomain = "acme"

[providers.pagerduty.europe]
token_env = "PAGERDUTY_EU_TOKEN"
api_url = "https://api.eu.pagerduty.com"
subdomain = "acme-eu"
```

PagerDuty profiles may also set `auth_type` and `from_email` when required by
the selected API operations.

## Operational Practices

- Use descriptive profile names such as `readonly`, `security_app`, or
  `production-eu`.
- Store secrets in a shell, CI secret store, or workload identity system.
- Give each environment variable the narrowest practical permissions.
- Keep target selectors in YAML so workflow scope remains reviewable.
- Use `anvil validate --auth --config-file targets.yaml` before a run.
- Avoid putting credential values in logs, task results, processor output, or
  profile names.
