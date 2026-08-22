# CLI

Anvil exposes four primary commands:

```console
anvil run       # Execute schema v2 provider-target YAML
anvil list      # List tasks, processors, and providers
anvil validate  # Inspect the environment or run focused validation
anvil results   # Query, process, or rerun completed results
```

Task dependencies are validated and resolved automatically by `validate` and
`run`; they do not require a separate command.

## Logging

Every command supports `--log-level` with `DEBUG`, `INFO`, `WARNING`, `ERROR`,
or `CRITICAL`.

```console
anvil run --config-file ./yaml/targets.yaml --log-level INFO
anvil validate --auth --config-file ./yaml/targets.yaml --log-level WARNING
```

## List Components

List installed component catalogs:

```console
anvil list --tasks
anvil list --processors
anvil list --providers
```

Task listings are grouped by universal and provider-specific packages. Extension
package sources are shown alongside stock components, and ambiguous duplicate
names are reported.

Show generated detail documentation for exactly one task or processor:

```console
anvil list --tasks count_vpc --detail
anvil list --processors html_report --detail
```

`--detail` is not supported with `--providers`.

## Validate

With no focused switches, `anvil validate` prints offline environment
diagnostics: Python and Anvil versions, optional provider dependency
availability, provider/task/processor discovery, local auth-source hints, and
result-path state. It does not call provider APIs or run tasks.

```console
anvil validate
```

Validate one or more schema v2 configuration files offline:

```console
anvil validate --config-file ./yaml/targets.yaml
anvil validate --config-file ./yaml/aws.yaml ./yaml/github.yaml
```

Offline configuration validation covers parsing, JSON Schema, semantic target
rules, provider/task compatibility, task dependencies, processor references,
and CLI selection overrides.

Run focused component validation:

```console
anvil validate --tasks
anvil validate --tasks count_vpc noop
anvil validate --processors html_report sarif_report
anvil validate --providers
anvil validate --providers aws github
```

`--tasks` and `--processors` validate discovery, signatures, and detail
documentation. `--providers` validates the provider component contract.

### Authentication

`--auth` performs provider-specific access checks for configured runnable
targets without executing tasks:

```console
anvil validate --auth --config-file ./yaml/targets.yaml
anvil validate --tasks --processors --providers --auth \
  --config-file ./yaml/targets.yaml
```

Authentication uses the configuration's provider credential model:

| Provider | `--auth` behavior |
| --- | --- |
| AWS | Creates the base boto3 session and checks STS identity |
| Azure | Loads the selected credential and acquires an Azure Resource Manager token |
| Cloudflare | Resolves credentials and validates SDK client construction; permissions are checked during discovery or execution |
| Datadog | Performs a live API/application-key validation for the configured site |
| GCP | Reports validation as deferred; credentials are built when runtime sessions are prepared |
| GitHub | Resolves token or App settings without making an API request |
| GitLab | Performs a live authenticated API request against the configured instance |
| PagerDuty | Resolves the token and validates REST client construction without making an API request |

Anvil caches equivalent authentication checks within the command and gives each
target its own result. Use `--quiet` in CI to suppress validation output and
rely on the exit code.

`--include` narrows auth checks to matching provider target IDs. `--exclude` is
valid only for discovery-based configurations. The two switches are mutually
exclusive.

## Run

Execute one schema v2 file:

```console
anvil run --config-file ./yaml/targets.yaml
```

Execute multiple files sequentially:

```console
anvil run --config-file ./yaml/aws.yaml ./yaml/azure.yaml ./yaml/github.yaml
```

Each file is an isolated run with its own output directory. The overall command
exits non-zero if any file fails.

Runtime selection and mode switches:

```console
# Narrow to provider execution-target IDs.
anvil run --config-file ./yaml/targets.yaml --include target-a target-b

# Exclude IDs from discovery-based targets.
anvil run --config-file ./yaml/targets.yaml --exclude target-c

# Force every selected task into dry-run mode.
anvil run --config-file ./yaml/targets.yaml --dry-run

# Add diagnostic phase timings to result JSON.
anvil run --config-file ./yaml/targets.yaml --benchmark
```

`--include` and `--exclude` are mutually exclusive. `--benchmark` adds detailed
phase timings to result JSON and can substantially increase output size.

Every run writes:

```text
results/
  <config-stem>/
    <run-id>/
      summary.json
      results.jsonl
      targets/
        <configured-target>.json
      reports/
        ...
```

`summary.json` contains the engine summary, `targets/` contains the full result
for each configured target, and `results.jsonl` contains flattened entity and
task records for querying and reruns.

## Processors

Processors turn completed results into reports or integration artifacts. List
and validate them without running cloud tasks:

```console
anvil list --processors
anvil validate --processors
```

Run a processor against a completed run directory:

```console
anvil results \
  --results-dir ./results/targets/2026-07-22T183012Z \
  --processor html_report \
  --output report.html
```

The run directory must contain `summary.json` and target JSON beneath
`targets/`. Relative processor output is resolved under that run's `reports/`
directory. Output paths are sanitized and kept beneath `reports/` even when the
input contains absolute or parent-directory segments.

Targets can also run processors automatically with `post_run`. By default they
run after successful targets; `run_on_failure: true` enables reports designed to
handle failed target results. Target-level output filenames are prefixed with
the configured target name, and collisions receive a numeric suffix.

Built-in processors include a self-contained `html_report` and a
`sarif_report` for `detect_` task results containing `sarif_findings`.

## Results

Without `--results-file`, `anvil results` searches for every `results.jsonl`
under `./results`.

```console
# Show every failure.
anvil results --status failed

# Show failed entities for one configured target.
anvil results --type entity --target production --status failed

# Select an AWS account, Azure subscription, GCP project, or GitHub target.
anvil results --entity 111111111111
anvil results --entity octo-org/platform-api

# Filter task records by provider location and task.
anvil results --type task --region us-east-1 --task count_vpc

# Choose columns and cap output.
anvil results --status failed \
  --fields target,entity_id,entity_type,region,task,error --limit 20

# Emit structured output.
anvil results --type task --status failed --jsonl
```

Query explicit result files:

```console
anvil results --status failed \
  --results-file ./results/aws/run-a/results.jsonl

anvil results --status failed \
  --results-file ./results/aws/run-a/results.jsonl \
                 ./results/github/run-b/results.jsonl
```

Available filters include `--type {entity,task}`, `--target`, `--entity`,
`--region`, `--task`, `--status`, `--fields`, and `--limit`. Use `--json` or
`--jsonl` for structured output. Task statuses are `success`, `error`,
`interrupted`, or `skipped`; `--status failed` matches `error` and
`interrupted`.

## Rerun Failures

`--rerun` loads the original config and infers the narrowest safe scope from
failed records. It selects failed configured targets and entities, narrows to
failed regions and tasks when task records are available, and includes required
task dependencies.

```console
anvil results --status failed \
  --results-file ./results/aws/run-a/results.jsonl \
  --rerun
```

Further narrow a rerun with `--target`, `--entity`, `--region`, or `--task`.
Use `--dry-run` to force dry-run behavior and `--benchmark` to collect rerun
timings. Report-shaping switches such as `--type`, `--fields`, `--limit`,
`--json`, and `--jsonl` cannot be combined with `--rerun`.
