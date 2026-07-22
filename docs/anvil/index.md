# Anvil

<p align="center">
  <img src="../images/anvil-logo-dark.png" alt="Anvil" width="236">
</p>

Anvil is a declarative, provider-aware execution engine for running Python tasks
across cloud and GitHub target fleets.

Describe the work in YAML, keep task logic in plain Python modules, and let the
engine handle authentication, target discovery, dependency ordering, bounded
concurrency, and structured results. AWS, Azure, GCP, and GitHub providers ship
with Anvil, and extension packages can add providers, tasks, and processors.

## Why Anvil?

Anvil is built for teams that need repeatable workflows to run consistently
across provider-specific execution targets and locations. Common use cases
include inventory, validation, enforcement, cleanup, security auditing, and
reporting.

## Core Capabilities

- Declarative orchestration with reusable YAML configuration.
- Provider-aware targets for AWS accounts, Azure subscriptions, GCP projects,
  and GitHub organizations or repositories.
- Explicit bounded concurrency for configured targets, resolved execution
  targets, and regions or locations.
- Shared provider discovery, authentication, and session reuse during a run.
- Task isolation through plain Python task modules with a `run(...)` function.
- Universal and provider-specific tasks plus extension-package components.
- Structured task, execution-target, configured-target, and engine results.
- Dry-run behavior, dependency ordering, fail-fast controls, and rerun support.

## Installation

The base package includes AWS support:

```console
pip install anvil
```

Install the SDK extras needed by the configured providers:

```console
pip install "anvil[azure]"
pip install "anvil[gcp]"
pip install "anvil[github]"
```

## Repository Template

Create a dedicated task repository with the
[foundry-anvil-template](https://github.com/JSChronicles/foundry-anvil-template).
The template provides a ready project layout for custom tasks, YAML examples,
validation, and CI outside of the main Anvil repository.

## Next

- Read the [execution model](execution-model.md) for how Anvil prepares targets,
  authenticates, resolves provider targets, and schedules task work.
- Read the [CLI reference](cli.md) for the main command groups and result
  commands.
- Read the [task contract](task-contract.md) for custom task discovery,
  validation, and the `run(...)` interface.
- Read [configuration](configuration.md) for schema v2 provider targets,
  selection, location, concurrency, and metadata concepts.
- Read [examples](examples.md) for the repository template, GitHub Actions, and
  standalone multi-account script template.
