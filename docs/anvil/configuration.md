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

## Defaults and Limits

| Setting | Scope | Default | Constraint |
| --- | --- | --- | --- |
| `schema_version` | File | Required | Must be `2` |
| `max_parallel_targets` | File | `1` | Integer, minimum `1` |
| `name` | Target | Required | Non-empty and unique in the file |
| `provider.name` | Target | Required | Discovered provider name |
| `provider.mode` | Target | Required | Provider-supported mode |
| `provider.options` | Target | `{}` | Provider-owned mapping |
| `regions` | Target | Provider default | Non-empty unique strings when set |
| `tasks` | Target | `noop` | Non-empty task list when set |
| `max_workers` | Target | `10` | Integer from `1` through `100` |
| `max_parallel_regions` | Target | `1` | Integer from `1` through `4` |
| `fail_fast` | Target | `false` | Boolean |
| `dry_run` | Target | `false` | Boolean |
| `metadata` | Target or task | `{}` | Free-form mapping |

Unknown file, target, provider-block, task, dependency-reference, and processor
fields are rejected. Provider option keys remain provider-owned and are checked
during semantic validation.

## Provider Block

The `provider` block has three fields:

- `name`: discovered provider component name
- `mode`: provider-specific discovery or explicit-target mode
- `options`: provider-specific authentication and runtime options

Use `anvil list --providers` to inspect installed providers and
`anvil validate --providers` to validate their contracts.

| Provider | Modes | Execution targets |
| --- | --- | --- |
| AWS | `organization`, `accounts` | AWS accounts |
| Azure | `tenant`, `subscriptions` | Azure subscriptions |
| Cloudflare | `accounts`, `zones` | Cloudflare accounts or zones |
| Datadog | `organization` | One key-bound organization |
| GCP | `projects`; `organization` is reserved | GCP projects |
| GitHub | `organizations`, `repositories` | GitHub organizations or repositories |
| GitLab | `groups`, `projects` | GitLab groups or projects |
| PagerDuty | `account` | One PagerDuty account |

Discovery modes resolve execution targets from the provider. Explicit modes use
the IDs supplied through `include`. AWS `organization` and Azure `tenant`
discover child accounts or subscriptions. GCP `projects` can either discover
accessible projects when `include` is omitted or use explicit project IDs;
GCP `organization` currently raises a not-implemented error. GitHub
`organizations` accepts owner logins and either runs a dedicated organization
code search or discovers repositories beneath those owners; repository mode
uses `owner/repository` IDs. See the [provider reference](providers.md) for
options, authentication, selection rules, install extras, and examples for
every stock provider.

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

Explicit Azure subscription selection:

```yaml
schema_version: 2

targets:
  - name: azure-subscriptions
    provider:
      name: azure
      mode: subscriptions
      options: {}
    include:
      - 00000000-0000-0000-0000-000000000000
    regions:
      - eastus
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

Reusable Cloudflare, Datadog, GitHub, GitLab, and PagerDuty settings live in
provider-namespaced tables in `~/.anvil/config.toml`. See
[provider profiles](provider-profiles.md) for named/default profiles,
environment-variable references, multiple endpoints, and advanced examples.

## Selection

`include` and `exclude` are mutually exclusive lists of provider target IDs.
They can also be supplied as mutually exclusive `--include` or `--exclude` CLI
overrides.

- `include` narrows execution to explicit IDs or a subset of discovered IDs.
- `exclude` removes IDs from providers that support discovery filtering. It is
  not valid for explicit AWS, Azure, or GitHub repository targets, and GitHub
  does not currently support exclusion in either mode.
- Unknown discovered IDs produce a warning while valid targets continue.

The meaning of an ID is provider-specific. See
[selectors and regions](selectors-and-regions.md) for every provider mode,
AWS `management`/`payer` keywords, CLI narrowing, and identifier tips.

## Regions and Locations

`regions` is the provider location dimension passed to task execution:

- AWS uses enabled AWS Regions.
- Azure and GCP use provider-discovered cloud locations.
- Cloudflare, Datadog, GitHub, GitLab, and PagerDuty use `global`.

AWS, Azure, and GCP support provider-appropriate `all` and glob selectors. See
[selectors and regions](selectors-and-regions.md) for exact mode support and
availability behavior.

Tasks with target scope run once for an execution target using its first
resolved location. Region-scoped tasks run once per resolved location. The
provider declares which scopes it supports.

## Concurrency and Failure Controls

- `max_parallel_targets` limits concurrent configured targets for the YAML.
- `max_workers` limits concurrent execution targets within one configured
  target, such as accounts, subscriptions, projects, or repositories.
- `max_parallel_regions` limits concurrent regions or locations within one
  execution target.
- `fail_fast` cooperatively stops pending work after a task failure.

Increase concurrency gradually and benchmark with the provider APIs, target
count, location count, and task mix expected in production.

## Dry Run

`dry_run` is passed to every task. The engine does not automatically prevent a
provider mutation; task implementations must honor dry-run behavior. The CLI
`--dry-run` switch forces dry-run mode for the selected run or rerun.

## Tasks and Dependencies

Tasks are ordered declaratively and may depend on earlier invocations:

```yaml
tasks:
  - name: inventory
  - id: report_inventory
    name: report
    depends_on:
      - inventory
    dependency_data:
      inventory_result:
        task_id: inventory
        path: result
```

Anvil validates the dependency graph before execution. Normal dependents run
only after every dependency succeeds. `always_run` supports cleanup after
unsuccessful dependencies, and `dependency_data` selects complete task results
or nested values for consumers.

Task compatibility is based on package location. Universal tasks apply to every
provider; provider-specific task packages apply only to their provider. See the
[task contract](task-contract.md) for the callable interface and
[task workflows](task-workflows.md) for IDs, result sharing, recovery, and
scope-aware dependencies.

## Metadata

Target `metadata` supplies shared static task inputs. Task-level metadata is
recursively merged over it for that invocation. Use metadata for expected
policy values, branch names, search queries, runtime lists, or reporting
context; use `dependency_data` for values produced during the run.

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
processor when the target failed. See
[extension best practices](extension-best-practices.md#build-a-processor) to
create and package a processor.
