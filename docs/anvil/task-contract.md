# Task Contract

Anvil keeps AWS business logic in plain Python task modules while the engine
handles authentication, role assumption, dependency ordering, result
aggregation, and concurrency.

## Task Discovery

Tasks are discovered from two sources:

- Stock tasks shipped with Anvil under `anvil.tasks`.
- Plugin tasks registered through the `anvil.tasks` entry-point group.

Directories named `tasks/` are conventional only. They are not automatically
scanned unless the surrounding project registers tasks through the plugin
mechanism.

Once configured, custom tasks behave like stock tasks:

```yaml
tasks:
  - name: inventory
  - name: cleanup
    depends_on: [inventory]
```

## Runtime Contract

Each task module must define a callable `run` function. This is the minimum
interface required for Anvil to discover and execute a task.

```python
from anvil.actions import ActionRecorder

def run(
    *,
    account_id: str,
    account_alias: str,
    session,
    dry_run: bool,
    metadata: dict[str, object],
    actions: ActionRecorder,
) -> None:
    """
    Execute the task for one AWS account-region pair.
    """
```

## Arguments

- `account_id`: AWS account ID currently being processed.
- `account_alias`: Friendly name of the account.
- `session`: boto3-style session already scoped to the target account and
  region.
- `dry_run`: Indicates whether the task should make changes.
- `metadata`: Organization metadata defined in the configuration file.
- `actions`: Action recorder provided by Anvil for planned or completed work.

The return value is optional. Any returned data may be included in execution
results.

## Returned Results

A task can return any JSON-serializable value from `run()`. Returned data is the
native baseline for structured task output. It is stored under each task
result's `result` field and is useful for inventory, measurements, findings,
resource IDs, counts, timing, and other task-specific data.

```python
def run(
    *,
    account_id: str,
    account_alias: str,
    session,
    dry_run: bool,
    metadata: dict[str, object],
    actions=None,
) -> dict[str, object]:
    user_name = str(metadata["user_name"])
    iam = session.client("iam")
    groups = [
        group["GroupName"]
        for group in iam.list_groups_for_user(UserName=user_name)["Groups"]
    ]

    return {
        "user_name": user_name,
        "dry_run": dry_run,
        "groups": groups,
        "summary": {"groups": len(groups)},
    }
```

The returned value appears in the task result:

```json
{
  "task": "function_returned_results",
  "region": "us-east-1",
  "status": "success",
  "result": {
    "user_name": "example-user",
    "dry_run": false,
    "groups": ["Developers"],
    "summary": {
      "groups": 1
    }
  },
  "error": null
}
```

Returned data is also available in the flattened JSONL query artifact. See
[CLI results](cli.md#results) for query examples.

## ActionRecorder

Tasks can use Anvil-provided utilities to produce structured results.
`ActionRecorder` allows tasks to:

- record planned or executed actions
- produce structured output for reporting
- integrate with Anvil execution summaries

Using these utilities is not required, but it is recommended for tasks that
modify infrastructure or need richer audit output.

Record concise audit-level actions from a task:

```python
from anvil.actions import ActionRecorder

def run(
    *,
    account_id: str,
    account_alias: str,
    session,
    dry_run: bool,
    metadata: dict[str, object],
    actions: ActionRecorder,
) -> None:
    if dry_run:
        actions.record("(dry-run) Would validate account configuration")
    else:
        actions.record("Validated account configuration")
```

## Choosing a Result Channel

Use returned results when the task needs to report structured data:

- inventory lists
- counts and measurements
- validation findings
- resource IDs and metadata
- timing or diagnostic values

Use `ActionRecorder` when the task needs a concise audit trail:

- created, updated, or deleted resources
- skipped resources and decisions
- dry-run planned actions
- governance or cleanup outcomes

Production tasks may use both channels when that is useful.

See the [Anvil Results examples](https://github.com/JSChronicles/anvil/tree/main/examples/Results)
for complete returned-result and `ActionRecorder` examples.

## Task Validation

Anvil includes a task validation mode that checks discovered tasks for
structural correctness without executing them:

```console
anvil validate --tasks
```

Validation verifies that:

1. the task has a valid non-empty name
2. the task exposes a callable `run(...)` entrypoint
3. the `run(...)` signature includes required runtime parameters
4. the task does not use unsupported positional-only parameters
5. duplicate task names are rejected

Because this validation is structural, it does not perform AWS calls or execute
task logic.

Example validation failure:

```console
[ERROR] task validation failed:
  - task 'cleanup' is missing required run() parameters: ['account_alias']
  - task 'inventory' is missing required run() parameters: ['metadata']
```

Example validation success:

```console
[OK] all tasks are valid
```

## Dependency-Aware Execution

Tasks execute in dependency order within each account-region pair.

If a task depends on a failed earlier dependency, Anvil records that task as
blocked by dependency failure. Optional tasks can be skipped after dependency
failure without failing the entire account, while non-optional task failures
stop further execution for that region.
