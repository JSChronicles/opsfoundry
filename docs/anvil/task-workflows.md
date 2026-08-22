# Task Workflows and Result Sharing

Anvil tasks can form a dependency graph inside each configured target. A task
can wait for earlier tasks and select structured values from their
`TaskResult` objects without using global state or temporary files.

## Invocation IDs

`name` selects a discovered component. `id` identifies one configured use of
that component and defaults to `name` when omitted:

```yaml
tasks:
  - name: inventory_users
  - id: remove_departed_users
    name: remove_iam_user
```

If the same component appears more than once, every occurrence needs an
explicit, unique `id`. Dependencies always reference effective IDs, not a
component-name fallback.

## Ordering with `depends_on`

Declare direct dependencies with `depends_on`:

```yaml
tasks:
  - name: inventory_users
  - id: remove_departed_users
    name: remove_iam_user
    depends_on:
      - inventory_users
```

A normal dependent task runs only after every dependency succeeds. If a
dependency errors, is interrupted, or is skipped, the dependent task is
recorded as blocked.

## Static Metadata vs Runtime Dependency Data

Use `metadata` for operator-configured values known before the run. Target
metadata is recursively merged with task metadata, and task values take
precedence.

Use `dependency_data` for values produced during the run:

```yaml
tasks:
  - name: inventory_users

  - id: remove_departed_users
    name: remove_iam_user
    depends_on:
      - inventory_users
    metadata:
      reason: employment-ended
    dependency_data:
      users:
        task_id: inventory_users
        path: result.departed_user_ids
```

The consuming task receives the selected values in its keyword-only
`dependency_data` argument:

```python
def run(*, metadata: dict[str, object], dependency_data: dict[str, object], **kwargs):
    reason = metadata["reason"]
    users = dependency_data["users"]
    return {"reason": reason, "selected_count": len(users)}
```

Treat `metadata` and `dependency_data` as read-only. Anvil copies their nested
mappings and lists for each invocation.

## Dependency Paths

Each `dependency_data` entry names a direct dependency and can select:

| Configuration | Value passed to the consumer |
| --- | --- |
| Omit `path` | Complete producer `TaskResult` |
| `path: result` | Producer return value |
| `path: result.users` | Nested mapping value |
| `path: status` | Producer status |
| `path: error` | Producer error value |
| `path: actions` | Producer recorded actions |

Paths use dotted mapping fields. List indexing is not supported. Existing null
values are valid; a missing path causes the consumer to fail before its
`run()` function is called.

## Cleanup with `always_run`

Set `always_run: true` on a task with at least one dependency when cleanup or
restoration must run after dependencies settle, including unsuccessful ones:

```yaml
tasks:
  - id: detach_guardrails
    name: reconcile_config_guardrails
    metadata:
      attachment_state: absent

  - id: restore_guardrails
    name: reconcile_config_guardrails
    depends_on:
      - detach_guardrails
    always_run: true
    metadata:
      attachment_state: present
    dependency_data:
      attachments:
        task_id: detach_guardrails
        path: result.attachments
```

`always_run` changes eligibility; it does not erase an upstream failure.

## Preserve Recovery Data on Failure

A task that partially mutates resources can raise `TaskExecutionError` with a
JSON-serializable partial result. Downstream cleanup can select that recovery
data:

```python
from anvil.task_errors import TaskExecutionError


raise TaskExecutionError(
    "Mutation partially failed",
    partial_result={"attachments": detached_attachments},
)
```

Pair this with an `always_run` cleanup task that selects
`result.attachments`.

## Scope-Aware Fan-Out and Fan-In

Task scope is declared in Python with `TASK_SCOPE`; it is not configured in
YAML. Region-scoped tasks run for every resolved location. Target-scoped tasks
run once per execution target. Providers that support `configured_target`
tasks can also run once for the complete YAML target.

When scopes differ, Anvil maps dependency results across the execution graph:

- a configured-target producer can fan out to target or region consumers;
- region results can fan in to a configured-target consumer;
- ordering remains deterministic even when regions run concurrently.

The selected value is a single object when one producer invocation maps to the
consumer. It is a list when multiple producer invocations fan in. For example,
a configured-target consumer selecting `path: result` from a region task
receives one return value per mapped region in deterministic plan order. Write
consumers to validate both the expected container shape and the selected value
types.

See the complete
[configured-target cleanup workflow](https://github.com/JSChronicles/anvil/blob/main/examples/33-aws-config-cleanup-workflow.yaml)
for fan-out, fan-in, recovery data, and `run_on_failure` reporting.

## Workflow Checklist

- Give repeated task components explicit IDs.
- Depend only on tasks whose completion is required.
- Select only the result fields a consumer needs.
- Keep static operator intent in `metadata` and runtime values in
  `dependency_data`.
- Make every returned value JSON-serializable.
- Use `always_run` only for genuine finalization or recovery.
- Validate the graph with `anvil validate --config-file workflow.yaml`.
