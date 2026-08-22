# Selectors and Regions

Anvil separates **target selection** from **location selection**:

- `include` and `exclude` select provider resources such as accounts, projects,
  repositories, zones, or groups.
- `regions` selects the provider locations where compatible tasks run.

They use different rules. Region values may support glob patterns; target IDs
are exact values unless a provider documents a special keyword.

## Include and Exclude

`include` and `exclude` are mutually exclusive. Their meaning depends on the
provider mode:

| Provider mode | Selection behavior |
| --- | --- |
| AWS `organization` | Optional `include` or `exclude` of discovered account IDs |
| AWS `accounts` | `include` is required; `exclude` is not allowed |
| Azure `tenant` | Optional `include` or `exclude` of discovered subscriptions |
| Azure `subscriptions` | `include` is required; `exclude` is not allowed |
| Cloudflare `accounts`, `zones` | Omit both to discover, use `include` for exact IDs, or `exclude` from discovery |
| Datadog `organization` | Target filtering is not supported; configure one target per organization |
| GCP `projects` | Omit `include` to discover projects; discovery supports `exclude`; explicit `include` does not |
| GitHub `organizations`, `repositories` | `include` is required; `exclude` is not allowed |
| GitLab `groups`, `projects` | Omit both to discover, use `include` for exact IDs or paths, or `exclude` from discovery |
| PagerDuty `account` | Target filtering is not supported; configure one target per account |

Use strings for identifiers that are numeric or may have leading zeros:

```yaml
include:
  - '111111111111'
```

Target selection does not support globbing. Use exact provider IDs, names, or
paths in the shape required by the selected mode.

## AWS Management-Account Keywords

AWS `organization` mode recognizes `management` and `payer` in `include` and
`exclude`. Both values resolve to the organization management account, and the
comparison is case-insensitive:

```yaml
provider:
  name: aws
  mode: organization
  options:
    profile: security-audit
    role_name: OrganizationAccountAccessRole
include:
  - management
regions:
  - us-east-1
# ...
```

These keywords are not valid in AWS `accounts` mode.

## CLI Selection Overrides

The `--include` and `--exclude` CLI options follow provider selection rules.
When YAML already has an explicit `include`, a CLI `--include` can only narrow
that configured set; it cannot add new target IDs.

```console
anvil run --config-file targets.yaml --include 111111111111 222222222222
```

Do not pass `--exclude` when the YAML contains an explicit `include` or the
provider mode requires explicit targets.

## Explicit Regions

Use concrete values when exact coverage matters:

```yaml
regions:
  - us-east-1
  - us-west-2
```

Cloudflare, Datadog, GitHub, GitLab, and PagerDuty use the provider-neutral
location `global`:

```yaml
regions:
  - global
```

## The `all` Region Selector

Use lowercase `all` to select every available location discovered by a
provider. It must be the only entry:

```yaml
regions:
  - all
```

`all` is supported for AWS `organization`, Azure `tenant` and `subscriptions`,
and GCP `projects`. It is not a target-selection keyword and cannot appear in
`include` or `exclude`.

## Region Globs

AWS, Azure, and GCP discovery-backed locations support `*` glob patterns:

```yaml
regions:
  - us-*
  - eu-*
```

Globs can be mixed with concrete locations:

```yaml
regions:
  - us-*
  - ca-central-1
```

Anvil resolves selectors against provider-discovered locations. A glob that
matches no known location is an error. AWS runs only enabled Regions, Azure
uses locations available to the subscription, and GCP uses Compute regions
whose status is `UP`.

## Quick Tips

- Omit `regions` to use the provider default.
- Prefer explicit regions for tightly controlled production workflows.
- Use `all` for intentional full coverage, not as a convenience default.
- Start with one or two regions before increasing `max_parallel_regions`.
- Quote account IDs and other numeric-looking identifiers.
- Use `anvil validate --config-file targets.yaml` before authentication or a run.
- Use the [provider reference](providers.md) to confirm mode-specific selector
  rules.
