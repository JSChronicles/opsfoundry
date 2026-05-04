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

## ActionRecorder

Tasks can use Anvil-provided utilities to produce structured results.
`ActionRecorder` allows tasks to:

- record planned or executed actions
- produce structured output for reporting
- integrate with Anvil execution summaries

Using these utilities is not required, but it is recommended for tasks that
modify infrastructure or need richer audit output.

## Task Validation

Anvil includes a task validation mode that checks discovered tasks for
structural correctness without executing them:

```console
anvil tasks validate
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
