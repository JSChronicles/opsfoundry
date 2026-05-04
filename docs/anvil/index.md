# Anvil

<p align="center">
  <img src="../images/anvil.png" alt="Anvil" width="160">
</p>

Anvil is a declarative AWS execution engine for running Python tasks across
large account and region fleets.

Describe the work in YAML, keep task logic in plain Python modules, and let the
engine handle authentication, role assumption, dependency ordering, bounded
concurrency, and structured results. The goal is repeatable AWS work without
turning orchestration into one-off custom scripts.

## Why Anvil?

Anvil is built for teams that need repeatable AWS workflows to run consistently
across organizations, accounts, and regions. Common use cases include inventory,
validation, enforcement, cleanup, and reporting.

## Core Capabilities

- Declarative orchestration with reusable YAML configuration.
- Multi-account and multi-organization execution by default.
- Explicit bounded concurrency for targets, accounts, and regions.
- Shared organization discovery and session reuse during a run.
- Task isolation through plain Python task modules with a `run(...)` function.
- Stock tasks plus project-local tasks registered as plugins.
- Structured task, account, target, and engine-level results.
- Dry-run behavior, dependency ordering, fail-fast controls, and rerun support.

## Repository Template

Create a dedicated task repository with the
[foundry-anvil-template](https://github.com/JSChronicles/foundry-anvil-template).
The template provides a ready project layout for custom tasks, YAML examples,
validation, and CI outside of the main Anvil repository.

## Next

- Read the [execution model](execution-model.md) for how Anvil prepares targets,
  authenticates, assumes roles, and schedules account-region work.
- Read the [CLI reference](cli.md) for the main command groups and result
  commands.
- Read the [task contract](task-contract.md) for custom task discovery,
  validation, and the `run(...)` interface.
- Read [configuration](configuration.md) for target, region, concurrency, and
  metadata concepts.
- Read [examples](examples.md) for the repository template, GitHub Actions, and
  standalone multi-account script template.
