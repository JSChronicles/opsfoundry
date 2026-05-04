# CLI

Anvil exposes these primary command groups:

```console
anvil auth ...
anvil graph ...
anvil results ...
anvil tasks ...
anvil run ...
```

## Logging Verbosity

The `run`, `auth check`, and `graph` commands support `--log-level` to control
console output verbosity.

Supported values:

- `DEBUG`
- `INFO`
- `WARNING`
- `ERROR`
- `CRITICAL`

Examples:

```console
anvil run --config-file ./yaml/orgs.yaml --log-level ERROR
anvil auth check --config-file ./yaml/orgs.yaml --log-level WARNING
anvil graph --config-file ./yaml/orgs.yaml --log-level INFO
```

## Authentication

Authentication checks validate AWS credentials and access without executing
tasks.

```console
anvil auth check --help
```

Authenticate credentials from an organization file:

```console
anvil auth check --config-file ./yaml/orgs.yaml
```

Suppress output and rely on the exit code, which is useful for CI:

```console
anvil auth check --config-file orgs.yaml --quiet
```

### What Auth Check Does

For each configured organization, Anvil:

1. Infers the likely authentication source.
2. Creates a boto3 session.
3. Calls AWS STS `GetCallerIdentity`.
4. Records a structured result with status, source, timing, message, and
   optional remediation guidance.

`auth check` is a lightweight preflight validation step, not a full execution
run.

### Supported Authentication Sources

Anvil can classify authentication as:

- `SSO`
- `Profile static`
- `Profile assume role`
- `Environment`
- `OIDC`
- `Unknown`

This classification improves failure reporting and remediation guidance.

Common normalized failures include:

- AWS profile not found
- no AWS credentials available
- AWS SSO session is invalid or expired
- AWS credentials have expired
- access denied when calling AWS
- unexpected authentication error

## Graph

Display the resolved task dependency graph for an organization configuration:

```console
anvil graph --help
anvil graph --config-file ./examples/07-optional-task-semantics.yaml
```

Output graph results as JSON:

```console
anvil graph --config-file ./examples/07-optional-task-semantics.yaml --json
```

Example graph output:

```console
Execution Graph (optional-semantics-org)
----------------------------------------
inventory
`-- reporting
    `-- cleanup
```

Example JSON output:

```json
{
  "organization": "optional-semantics-org",
  "tasks": [
    {
      "name": "inventory",
      "depends_on": []
    },
    {
      "name": "reporting",
      "depends_on": ["inventory"]
    },
    {
      "name": "cleanup",
      "depends_on": ["reporting"]
    }
  ]
}
```

## Task Management

List all available stock and user-defined tasks:

```console
anvil tasks list
```

Example output:

```console
Available tasks:
plugin: my-test-project:
  - hello
  - test

stock:
  - compare_asg_to_cluster_instances
  - get_aws_inline_policies
  - get_organization_structure
  - noop
  - noop_fail
  - remove_iam_user
  - remove_missing_group_assignments
```

Validate discovered tasks:

```console
anvil tasks validate
```

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

See [Task Contract](task-contract.md) for discovery and validation details.

## Run

Execute configured organizations and accounts from one or more YAML files:

```console
anvil run --help
anvil run --config-file ./yaml/orgs.yaml
```

Run multiple YAML files sequentially:

```console
anvil run --config-file ./yaml/orgs.yaml ./yaml/orgs2.yaml ./yaml/orgs3.yaml
```

Each YAML remains an isolated run with its own summary file, and the overall
command exits non-zero if any YAML run fails.

Organization configs write per-target full results under `organizations/`:

```text
results/
  <config-stem>/
    <run-id>/
      summary.json
      results.jsonl
      organizations/
        <organization>.json
```

Account-group configs use `account-groups/`:

```text
results/
  <config-stem>/
    <run-id>/
      summary.json
      results.jsonl
      account-groups/
        <account-group>.json
```

Use `--benchmark` only for performance investigations. It adds engine, target,
account, region, and result-write timing details that can dramatically increase
output size on large account, region, or task runs.

## Results

Runs write full JSON result files and flattened JSONL records for quick
filtering:

```text
./results/{config-stem}/{run-id}/results.jsonl
```

Common queries:

```console
# Show every failure under ./results.
anvil results --status failed

# Show failures for one organization or account-group target.
anvil results --target prod --status failed

# Show failed account records only.
anvil results --type account --status failed

# Show task records for one task name.
anvil results --type task --task count_vpcs

# Show task records for one AWS region.
anvil results --type task --region us-east-1

# Show a compact failure view with selected fields and a row limit.
anvil results --status failed --fields account_id,region,task,error --limit 20

# Emit failed task records as JSONL.
anvil results --type task --status failed --jsonl
```

Advanced queries:

```console
# Query one explicit run results file.
anvil results --status failed --results-file ./results/orgs/2026-05-01T183012Z/results.jsonl

# Query multiple explicit run results files in one command.
anvil results --status failed --results-file ./results/orgs/run-a/results.jsonl ./results/accounts/run-b/results.jsonl

# Filter one task in one target and print selected fields.
anvil results --type task --target prod --task count_vpcs --fields account_id,region,status,error

# Show failure rows with target, account, region, task, and error context.
anvil results --status failed --fields record_type,target,account_id,region,task,error
```

## Rerun Failures

`--rerun` infers rerun scope from result records. It reloads the original config,
reruns only matching failed accounts, narrows to failed regions and tasks when
task-level failures are available, and includes required task dependencies
automatically.

Use scope filters such as `--target`, `--account`, `--region`, and `--task` to
limit a rerun even further. Report-shaping flags such as `--type`, `--fields`,
`--limit`, `--json`, and `--jsonl` are not supported with `--rerun`.

```console
# Rerun failures from one explicit run results file.
anvil results --status failed --results-file ./results/orgs/2026-05-01T183012Z/results.jsonl --rerun

# Rerun failures from multiple explicit run results files in one command.
anvil results --status failed --results-file ./results/orgs/run-a/results.jsonl ./results/accounts/run-b/results.jsonl --rerun
```
