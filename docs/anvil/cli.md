# CLI

Anvil exposes these primary command groups:

```console
anvil auth ...
anvil graph ...
anvil results ...
anvil tasks ...
anvil run ...
```

## Logging

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
anvil auth check --config-file ./yaml/orgs.yaml
```

Use `--quiet` to suppress output and rely on the exit code in CI:

```console
anvil auth check --config-file orgs.yaml --quiet
```

## Graph

Display the resolved task dependency graph for a configuration:

```console
anvil graph --config-file ./examples/07-optional-task-semantics.yaml
```

Emit graph output as JSON:

```console
anvil graph --config-file ./examples/07-optional-task-semantics.yaml --json
```

## Tasks

List stock and user-defined tasks:

```console
anvil tasks list
```

Validate discovered tasks:

```console
anvil tasks validate
```

Task validation checks that each task has a valid name, exposes a callable
`run(...)` entrypoint, includes the required runtime parameters, avoids
unsupported positional-only parameters, and does not duplicate another task
name.

## Run

Execute configured organizations and accounts from one or more YAML files:

```console
anvil run --config-file ./yaml/orgs.yaml
```

Run multiple YAML files sequentially:

```console
anvil run --config-file ./yaml/orgs.yaml ./yaml/orgs2.yaml ./yaml/orgs3.yaml
```

Anvil writes one run-scoped result directory per YAML input:

```text
results/
  <config-stem>/
    <run-id>/
      summary.json
      results.jsonl
      organizations/
        <organization>.json
```

Account-group configs use `account-groups/` for per-target JSON files:

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
account, region, and result-write timing details that can significantly increase
output size on large runs.

## Results

Anvil writes full JSON result files and a flattened JSONL query file:

```text
./results/{config-stem}/{run-id}/results.jsonl
```

Common queries:

```console
anvil results --status failed
anvil results --target prod --status failed
anvil results --type account --status failed
anvil results --type task --task count_vpcs
anvil results --type task --region us-east-1
anvil results --status failed --fields account_id,region,task,error --limit 20
anvil results --type task --status failed --jsonl
```

Rerun failures from a result file:

```console
anvil results --status failed --results-file ./results/orgs/run-a/results.jsonl --rerun
```

`--rerun` infers scope from result records, reloads the original config, reruns
matching failed accounts, narrows to failed regions and tasks where possible,
and includes required task dependencies automatically.
