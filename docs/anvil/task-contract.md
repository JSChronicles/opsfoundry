# Task Contract

Anvil keeps provider business logic in plain Python task modules while the
engine handles provider authentication, target resolution, dependency ordering,
bounded concurrency, and result aggregation.

## Task Packages and Discovery

Task compatibility is determined by the package that contributes the module:

- `anvil.providers.tasks.<task>` is universal.
- `anvil.providers.aws.tasks.<task>` is AWS-only.
- `anvil.providers.azure.tasks.<task>` is Azure-only.
- `anvil.providers.gcp.tasks.<task>` is GCP-only.
- `anvil.providers.github.tasks.<task>` is GitHub-only.

Third-party packages register task package roots through entry points:

```toml
[project.entry-points."anvil.providers.tasks"]
universal-tasks = "company_anvil.tasks"

[project.entry-points."anvil.providers.aws.tasks"]
aws-tasks = "company_anvil.aws_tasks"

[project.entry-points."anvil.processors"]
processors = "company_anvil.processors"

[project.entry-points."anvil.provider_packages"]
providers = "company_anvil.providers"
```

Each public Python filename is the task name. Discovery records component names
and sources without importing every implementation; Anvil imports only selected
tasks during normal execution. Duplicate applicable task names are rejected as
ambiguous and report every conflicting source.

For a provider collection, each immediate child package is a provider and must
expose `create_provider_instance()`. Processor package roots use the same
public-module discovery pattern as task packages.

Use these commands to inspect and validate discovery:

```console
anvil list --tasks
anvil list --tasks count_vpc --detail
anvil validate --tasks
anvil validate --tasks count_vpc noop
```

## Runtime Contract

Every task module defines a callable `run(...)` function. The provider-neutral
runtime keyword arguments are:

```python
from anvil.actions import ActionRecorder


def run(
    *,
    provider: str,
    execution_target_id: str,
    execution_target_name: str,
    execution_target_type: str,
    region: str,
    session,
    dry_run: bool,
    metadata: dict[str, object],
    actions: ActionRecorder,
) -> dict[str, object] | None:
    """Run the task for one provider execution target and location."""
```

- `provider`: selected provider name.
- `execution_target_id`: provider-specific ID, such as an account,
  subscription, project, owner, or repository ID.
- `execution_target_name`: provider-supplied display name.
- `execution_target_type`: provider-specific type such as `account`,
  `subscription`, `project`, `organization`, or `repository`.
- `region`: current provider region or location; GitHub uses `global`.
- `session`: provider session scoped to the current target and location.
- `dry_run`: whether task mutations should be suppressed.
- `metadata`: target metadata from the YAML configuration.
- `actions`: recorder for planned or completed audit actions.

Anvil invokes tasks with keyword arguments. A task may accept `**kwargs`, but
explicit parameters make the contract and generated detail output clearer.
Positional-only parameters are unsupported.

Task code can also declare a `task_context` parameter to receive the complete
immutable `TaskCallContext`; the nine arguments above remain the required
public contract unless `**kwargs` is accepted.

## Session Objects

The `session` interface is provider-specific:

- AWS tasks receive a boto3-compatible session with lazy client caching for the
  current account and region.
- Azure tasks receive an Azure session containing the credential,
  subscription ID, and location.
- GCP tasks receive a GCP session containing credentials, project ID, quota
  project, and region.
- GitHub tasks receive a GitHub session/client context for the current owner or
  repository.

Universal tasks should avoid assuming a provider SDK unless they branch on
`provider`. Provider-specific tasks should validate the execution target type
they require and provide actionable dependency errors.

## Task Scope

Tasks are region-scoped by default. A module can opt into target scope:

```python
TASK_SCOPE = "target"
```

A region-scoped task runs once per resolved region or location. A target-scoped
task runs once per execution target using its first resolved location. A task
can run only when the selected provider advertises support for that scope; AWS
currently supports region scope, while Azure, GCP, and GitHub support both
region and target scope.

## Detail Documentation

Every task needs detail documentation. Add a Google-style docstring to
`run(...)`; a module docstring is accepted as a fallback. This text powers
`anvil list --tasks <name> --detail` and is checked by task validation.

Document the operation, provider and scope assumptions, metadata keys, return
shape, and important exceptions. Never include credentials or secret values in
task detail text or results.

## Returned Results

A task may return any JSON-serializable value. Anvil stores it in the task
result's `result` field and includes it in flattened JSONL output.

```python
def run(
    *,
    provider: str,
    execution_target_id: str,
    execution_target_name: str,
    execution_target_type: str,
    region: str,
    session,
    dry_run: bool,
    metadata: dict[str, object],
    actions,
) -> dict[str, object]:
    client = session.client("ec2")
    vpc_ids = [
        vpc["VpcId"]
        for page in client.get_paginator("describe_vpcs").paginate()
        for vpc in page.get("Vpcs", [])
    ]
    return {
        "account_id": execution_target_id,
        "region": region,
        "vpc_count": len(vpc_ids),
        "vpc_ids": vpc_ids,
    }
```

Returned results work best for inventory, counts, findings, identifiers,
measurements, and other structured task data.

## ActionRecorder

Use `ActionRecorder` for a concise audit trail of planned or completed work:

```python
if dry_run:
    actions.record("(dry-run) Would update the repository setting")
else:
    actions.record("Updated the repository setting")
```

Tasks may use returned data and recorded actions together. Dry-run-aware
mutation tasks should record the planned operation without issuing the provider
mutation.

## Validation

`anvil validate --tasks` performs structural validation without running tasks
or calling provider APIs. It verifies:

1. task names are non-empty and unambiguous
2. modules expose a callable `run(...)`
3. the runtime signature accepts all required keyword arguments
4. positional-only parameters are absent
5. detail documentation is present

Configuration validation additionally checks that each configured task is
compatible with the selected provider and task scope.

## Dependency-Aware Execution

Tasks execute in dependency order within each execution-target/location stream.
If a dependency fails, dependent tasks are recorded as blocked. Optional task
failures are recorded without necessarily failing the execution target;
non-optional failures stop further work for that stream and participate in
fail-fast cancellation.
