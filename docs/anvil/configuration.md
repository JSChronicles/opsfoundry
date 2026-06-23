# Configuration

Anvil uses YAML to describe what should run, where it should run, and how much
parallelism is allowed.

## Target Scope

Targets can represent AWS Organizations or explicit account groups.

Organization targets support discovery-driven execution:

- discover active accounts
- discover enabled regions
- validate configured regions against enabled regions
- apply include/exclude account filters
- assume roles into member accounts

Explicit account targets use account lists supplied by configuration rather than
organization discovery.

## Per-Target Settings

Each target can define its own:

- AWS profile
- target regions
- role name
- include/exclude account filters
- worker concurrency
- region concurrency
- dry-run behavior
- fail-fast behavior
- task definitions
- metadata

This lets one YAML coordinate multiple AWS environments without forcing them
into a shared credential or runtime model.

## Target-Level Concurrency

`max_parallel_targets` bounds how many configured targets are allowed to prepare
or execute at once.

Targets that resolve to the same AWS organization can reuse organization
discovery results during a run. Concurrent preparation for the same organization
waits for in-flight discovery rather than issuing duplicate discovery calls.

## Account Concurrency

`max_workers` controls how many account executions may run at the same time for
one target.

This is the main concurrency control for large account fleets. Increase it to
speed up broad account-level workflows, but benchmark against the same task mix
and account count you expect in production.

## Region Concurrency

`max_parallel_regions` controls how many regions may execute concurrently inside
one account. Supported values are `1` through `4`.

Use region parallelism when each region has enough independent work to benefit
from overlap. For lightweight describe/list workloads, region parallelism can
increase AWS API pressure enough that each regional call slows down.

## Region Selection

Anvil validates configured regions against enabled AWS regions for the
organization. It only executes in the effective configured regions that remain
after validation.

Region selection may include explicit regions as well as broader selections such
as `all` or glob-like patterns, depending on the configuration.

Example explicit region selection:

```yaml
targets:
  - name: prod
    type: organization
    profile: prod-admin
    regions:
      - us-east-1
      - us-west-2
```

Example broad region selection:

```yaml
targets:
  - name: prod
    type: organization
    profile: prod-admin
    regions:
      - us-*
```

## Account Filters

Organization targets can narrow discovered accounts through include/exclude
filters.

If an include or exclude list references unknown account IDs, Anvil warns but
continues with valid discovered accounts that remain. This helps catch stale
configuration without turning harmless selection drift into a hard failure.

Example include filter:

```yaml
targets:
  - name: prod
    type: organization
    profile: prod-admin
    include:
      - "111111111111"
      - "222222222222"
```

Example exclude filter:

```yaml
targets:
  - name: prod
    type: organization
    profile: prod-admin
    exclude:
      - "999999999999"
```

## Dry Run

Dry-run behavior is passed into task execution as the `dry_run` argument. Tasks
are responsible for respecting it. This keeps write behavior explicit at the
task boundary while allowing the engine to carry the dry-run state consistently
across every account and region.

## Metadata

Target metadata is passed into tasks through the `metadata` argument. Use it for
configuration values that task logic needs but that should stay outside the task
code, such as policy choices, environment labels, expected settings, or
reporting context.

## Task Graph

Tasks are configured declaratively and may declare dependencies:

```yaml
tasks:
  - name: inventory
  - name: reporting
    depends_on: [inventory]
  - name: cleanup
    depends_on: [reporting]
```

Anvil resolves the task graph before execution. During execution, each
account-region pair runs tasks in dependency order.
