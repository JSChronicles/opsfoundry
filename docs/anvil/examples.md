# Examples

This page collects practical examples and reference patterns for running Anvil.

## GitHub Actions

For a complete GitHub Actions example that runs Anvil with AWS OIDC and uploads
generated JSON results as workflow artifacts, see
[examples/github-actions](https://github.com/JSChronicles/anvil/blob/main/examples/github-actions/README.md).

## Repository Template

Create a dedicated task repository using the
[foundry-anvil-template](https://github.com/JSChronicles/foundry-anvil-template).

The template provides a ready project layout for custom tasks, YAML examples,
validation, and CI outside of the main Anvil repository.

## Standalone Multi-Account Script Template

If you do not need the full Anvil framework and only want a small starting point
for AWS Organization tasks, see the
[standalone multi-account script template](https://github.com/JSChronicles/anvil/blob/main/templates/multi_aws_account_task_template.py).

The template provides:

- AWS Organizations account discovery
- active-account filtering with `--include` and `--exclude`
- parallel per-account execution
- multiple regions per account
- assume-role handling for member accounts
- dry-run support
- JSON result output

Replace the internals of `account_task()` with your own per-account logic and
adapt the example arguments as needed.

## Result Queries

Runs write flattened JSONL records for quick filtering:

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

```console
# Rerun failures from one explicit run results file.
anvil results --status failed --results-file ./results/orgs/2026-05-01T183012Z/results.jsonl --rerun

# Rerun failures from multiple explicit run results files in one command.
anvil results --status failed --results-file ./results/orgs/run-a/results.jsonl ./results/accounts/run-b/results.jsonl --rerun
```

Use scope filters such as `--target`, `--account`, `--region`, and `--task` to
limit a rerun even further. Report-shaping flags such as `--type`, `--fields`,
`--limit`, `--json`, and `--jsonl` are not supported with `--rerun`.

## Benchmark Notes

To measure concurrency behavior, Anvil was tested across 3 organizations with a
combined 260 accounts using the `count_vpc` task.

The fastest measured run in this benchmark completed 260 accounts in about
1m 35s for 1 region, compared with a 3h 15m manual sequential estimate at 45
seconds per account. With 2 regions, the parallel account run completed in about
2m 48s.

Use `--benchmark` only for performance investigations. It adds engine, target,
account, region, and result-write timing details to result JSON, which can
dramatically increase output size on large account, region, or task runs.
