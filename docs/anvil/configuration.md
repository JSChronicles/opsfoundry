# Configuration

Anvil schema v2 YAML describes which provider targets to resolve, which tasks to
run, and how execution should be bounded.

Every file starts with `schema_version: 2` and a non-empty `targets` list:

```yaml
schema_version: 2
max_parallel_targets: 2

targets:
  - name: aws-inventory
    provider:
      name: aws
      mode: organization
      options:
        profile: security-audit
        role_name: OrganizationAccountAccessRole
    regions:
      - us-east-1
    tasks:
      - name: count_vpc
```

`max_parallel_targets` limits how many configured targets can prepare or execute
at once. Each target then has its own provider, selection, concurrency, task,
dry-run, fail-fast, processor, and metadata settings.

## Provider Block

The `provider` block has three fields:

- `name`: provider component name, such as `aws`, `azure`, `gcp`, or `github`
- `mode`: provider-specific discovery or explicit-target mode
- `options`: provider-specific authentication and runtime options

Use `anvil list --providers` to inspect installed providers and
`anvil validate --providers` to validate their contracts.

| Provider | Modes | Execution targets | Common options |
| --- | --- | --- | --- |
| AWS | `organization`, `accounts` | AWS accounts | `profile`, `role_name` |
| Azure | `tenant`, `subscriptions` | Azure subscriptions | `tenant_id`, `client_id`, `client_secret` |
| GCP | `projects`; `organization` is reserved but not implemented | GCP projects | `credentials_path`, `quota_project_id`; `organization_id` is reserved for organization mode |
| GitHub | `organizations`, `repositories` | GitHub organizations or repositories | `profile`, `token_env`, `app_id`, `private_key_env`, `private_key_path`, `api_url`, `api_version` |

Discovery modes resolve execution targets from the provider. Explicit modes use
the IDs supplied through `include`. AWS `organization` and Azure `tenant`
discover child accounts or subscriptions. GCP `projects` can either discover
accessible projects when `include` is omitted or use explicit project IDs;
GCP `organization` currently raises a not-implemented error. GitHub
`organizations` accepts owner logins and either runs a dedicated organization
code search or discovers repositories beneath those owners; repository mode
uses `owner/repository` IDs.

## Provider Examples

AWS organization discovery:

```yaml
schema_version: 2
targets:
  - name: aws-production
    provider:
      name: aws
      mode: organization
      options:
        profile: production-root
        role_name: OrganizationAccountAccessRole
    regions: [us-east-1, us-west-2]
    exclude: ["999999999999"]
    tasks:
      - name: count_vpc
```

Azure subscription discovery with the default Azure credential chain:

```yaml
schema_version: 2
targets:
  - name: azure-subscriptions
    provider:
      name: azure
      mode: tenant
      options: {}
    regions: [eastus]
    exclude:
      - 99999999-aaaa-bbbb-cccc-dddddddddddd
    tasks:
      - name: count_resource_groups
```

To use explicit Azure client-secret credentials, provide `tenant_id`,
`client_id`, and `client_secret` together. If `client_secret` is omitted, Anvil
uses `DefaultAzureCredential`; `client_id` can select a managed identity.

GCP projects with application-default credentials:

```yaml
schema_version: 2
targets:
  - name: gcp-projects
    provider:
      name: gcp
      mode: projects
      options:
        quota_project_id: anvil-billing-project
    regions: [us-central1]
    include: [anvil-dev-project, anvil-prod-project]
    tasks:
      - name: get_project_info
```

Set `credentials_path` to load a specific credentials file instead of using
Google application-default credentials.

GitHub repositories with a token stored in an environment variable:

```yaml
schema_version: 2
targets:
  - name: github-security
    provider:
      name: github
      mode: repositories
      options:
        token_env: GITHUB_TOKEN
    regions: [global]
    include:
      - octo-org/platform-api
      - octo-org/platform-web
    metadata:
      branch: main
    tasks:
      - name: audit_branch_protection
      - name: audit_repo_security_settings
```

GitHub also supports app authentication through `app_id` plus exactly one of
`private_key_env` or `private_key_path`. A named `profile` cannot be combined
with inline auth options.

GitHub profiles are TOML tables in `~/.github/config` by default. Set
`ANVIL_GITHUB_CONFIG` to use another file:

```toml
[default]
token_env = "GITHUB_TOKEN"

[security-app]
app_id = "12345"
private_key_env = "GITHUB_APP_PRIVATE_KEY"
```

Set `provider.options.profile` to select a named table. Without an explicit
profile or inline auth options, Anvil tries the `default` profile, then
`GITHUB_TOKEN`, `GH_TOKEN`, `.netrc`, and `gh auth token`. `api_url` and
`api_version` support GitHub Enterprise and explicit API-version settings.

## Selection

`include` and `exclude` are mutually exclusive lists of provider target IDs.
They can also be supplied as mutually exclusive `--include` or `--exclude` CLI
overrides.

- `include` narrows execution to explicit IDs or a subset of discovered IDs.
- `exclude` removes IDs from providers that support discovery filtering. It is
  not valid for explicit AWS, Azure, or GitHub repository targets, and GitHub
  does not currently support exclusion in either mode.
- Unknown discovered IDs produce a warning while valid targets continue.

The meaning of an ID is provider-specific: an AWS account ID, Azure subscription
ID, GCP project ID, GitHub owner login, or GitHub `owner/repository` name.

## Regions and Locations

`regions` is the provider location dimension passed to task execution:

- AWS uses enabled AWS regions and supports explicit values, `all`, and glob
  selectors such as `us-*`.
- Azure and GCP use cloud locations and can resolve location selectors.
- GitHub uses the provider default `global` location.

Tasks with target scope run once for an execution target using its first
resolved location. Region-scoped tasks run once per resolved location. The
provider declares which scopes it supports.

## Concurrency and Failure Controls

- `max_parallel_targets` limits concurrent configured targets for the YAML.
- `max_workers` limits concurrent execution targets within one configured
  target, such as accounts, subscriptions, projects, or repositories.
- `max_parallel_regions` limits concurrent regions or locations within one
  execution target.
- `fail_fast` cooperatively stops pending work after a non-optional failure.

Increase concurrency gradually and benchmark with the provider APIs, target
count, location count, and task mix expected in production.

## Dry Run

`dry_run` is passed to every task. The engine does not automatically prevent a
provider mutation; task implementations must honor dry-run behavior. The CLI
`--dry-run` switch forces dry-run mode for the selected run or rerun.

## Tasks and Dependencies

Tasks are ordered declaratively and may depend on earlier tasks:

```yaml
tasks:
  - name: inventory
  - name: report
    depends_on: [inventory]
  - name: cleanup
    depends_on: [inventory, report]
    optional: true
```

Anvil validates the dependency graph before execution. A failed dependency
blocks dependent tasks. A failed optional task is recorded without necessarily
failing the execution target.

Task compatibility is based on package location. Universal tasks apply to every
provider; provider-specific task packages apply only to their provider. See the
[task contract](task-contract.md) for package and runtime details.

## Metadata

The target `metadata` mapping is passed unchanged to each task. Use it for task
inputs such as expected policy values, branch names, search queries, runtime
lists, or reporting context.

## Post-Run Processors

Attach processors to a target with `post_run`:

```yaml
post_run:
  - processor: html_report
    output: inventory.html
    run_on_failure: true
```

Processor output is written beneath the run's `reports` directory. Processors
run after successful targets by default; `run_on_failure: true` also runs a
processor when the target failed.
