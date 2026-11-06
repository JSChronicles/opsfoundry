# Task Scopes

Task scope controls how many times Anvil invokes a task and which execution
target owns the result. Choose the scope from the provider resource boundary,
not from the number of regions in the configuration.

Scope is part of the Python task module. It is not a YAML option:

```python
# Omit TASK_SCOPE for the default, region scope.
TASK_SCOPE = "target"
```

## The Three Scopes

| Scope | Invocation boundary | Choose it when | Typical examples |
| --- | --- | --- | --- |
| `region` | Once per resolved execution target and location | The API endpoint, resource inventory, or desired action can differ by region or location | VPCs, subnets, Lambda functions |
| `target` | Once per resolved execution target, using its first resolved location and session | The resource is global to each account, subscription, project, organization, or repository | AWS IAM users and policies, Azure resource groups |
| `configured_target` | Once for the original configured YAML target | The provider expands one configuration into many execution targets, but the work belongs to the configuration owner or shared control plane | AWS Organizations and IAM Identity Center |

`region` is the safe default for regional resources. A task that operates on an
account-wide or provider-global API should declare `target`; otherwise it runs
again for every configured region. This is especially dangerous for destructive
tasks because the same account-wide mutation can be attempted more than once.

## How to Choose

Ask these questions in order:

1. **Can the result or action differ by provider location?** Use `region`.
2. **Is it global, but independently owned by every resolved execution target?**
   Use `target`.
3. **Does it belong to the original configured AWS target or a shared owner
   account, rather than every discovered account?** Use `configured_target`.
4. **Does one module combine resources with different answers?** Split it into
   separate tasks so each task has one correct scope.

For example, an AWS configuration that resolves two accounts and three regions
produces six `region` invocations, two `target` invocations, and one
`configured_target` invocation.

```text
configured YAML target                         configured_target: 1
├── account A                                  target: 1
│   ├── us-east-1                              region: 1
│   ├── us-west-2                              region: 1
│   └── eu-west-1                              region: 1
└── account B                                  target: 1
    ├── us-east-1                              region: 1
    ├── us-west-2                              region: 1
    └── eu-west-1                              region: 1
```

## Scope Examples

### Regional resource

`count_vpc` queries resources that are different in every AWS Region, so it
uses the default `region` scope:

```python
# No TASK_SCOPE declaration: region is the default.
```

### Account-wide resource

`remove_iam_user` changes IAM users that belong to an AWS account, not a
Region. It runs once for each resolved account:

```python
TASK_SCOPE = "target"
```

Although a target-scoped task receives `region`, it is the target's first
resolved location and is used to create the session. It does not make the task
regional. Document whether the underlying API ignores that location or uses it
only as an SDK endpoint.

### Configured-target control plane

`get_organization_structure`, `get_aws_sso_inline_policies`,
`remove_idc_user`, and `remove_missing_group_assignments` work with an AWS
organization or IAM Identity Center owner. They run once for the configured
target instead of once for every account discovered beneath it:

```python
TASK_SCOPE = "configured_target"
```

Do not use `configured_target` merely to reduce API calls. It changes execution
ownership and dependency mapping, and it can cause per-account work to be
skipped.

## Provider Support

| Provider | Supported task scopes |
| --- | --- |
| AWS | `region`, `target`, `configured_target` |
| Azure | `region`, `target` |
| Cloudflare | `region`, `target` |
| Datadog | `region`, `target` |
| GCP | `region`, `target` |
| GitHub | `region`, `target` |
| GitLab | `region`, `target` |
| PagerDuty | `region`, `target` |

Providers advertise their supported scopes. `anvil validate --tasks` rejects an
unknown scope or an incompatible task/provider combination, but validation
cannot determine whether a provider API is regional or global. That semantic
choice remains part of task design and review.

## Dependencies Across Scopes

Anvil maps task dependencies across the execution graph. A
`configured_target` result can fan out to target- or region-scoped consumers,
and region results can fan in to a `configured_target` consumer. A dependency
selection is a single object when one producer invocation maps to the consumer
and a list when several producer invocations map to it.

Keep this mapping in mind when changing an existing task's scope: invocation
count, result ownership, dependency shape, and retry behavior may all change.
See [Task Workflows](task-workflows.md#scope-aware-fan-out-and-fan-in) for the
dependency contract.

## Review Checklist

- Identify the provider API's real ownership boundary.
- Count expected invocations for a multi-target, multi-location configuration.
- Verify that destructive work cannot repeat across locations.
- Confirm the provider advertises the selected scope.
- Test dependency fan-out or fan-in when connected tasks use different scopes.
- Run `anvil validate --tasks` and a focused task test.
