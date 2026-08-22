# Anvil

<p align="center">
  <img src="../images/anvil-logo-dark.png" alt="Anvil" width="236">
</p>

Anvil is a declarative, provider-aware execution engine for running Python tasks
across cloud and service target fleets.

Describe the work in YAML, keep task logic in plain Python modules, and let the
engine handle authentication, target discovery, dependency ordering, bounded
concurrency, and structured results. AWS, Azure, Cloudflare, Datadog, GCP,
GitHub, GitLab, and PagerDuty providers ship with Anvil, and extension packages
can add providers, tasks, and processors.

## Why Anvil?

Anvil is built for teams that need repeatable workflows to run consistently
across provider-specific execution targets and locations. Common use cases
include inventory, validation, enforcement, cleanup, security auditing, and
reporting.

## Core Capabilities

- Declarative orchestration with reusable YAML configuration.
- Provider-aware targets for cloud accounts, subscriptions, projects,
  organizations, repositories, zones, groups, and SaaS accounts.
- Explicit bounded concurrency for configured targets, resolved execution
  targets, and regions or locations.
- Shared provider discovery, authentication, and session reuse during a run.
- Task isolation through plain Python task modules with a `run(...)` function.
- Universal and provider-specific tasks plus extension-package components.
- Structured task, execution-target, configured-target, and engine results.
- Dry-run behavior, dependency ordering, fail-fast controls, and rerun support.

## Installation

Anvil 0.31 requires Python 3.14. The base package includes AWS support:

```console
pip install anvil
```

Install the SDK extras needed by the configured providers:

```console
pip install "anvil[azure]"
pip install "anvil[cloudflare]"
pip install "anvil[datadog]"
pip install "anvil[gcp]"
pip install "anvil[github]"
pip install "anvil[gitlab]"
pip install "anvil[pagerduty]"
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
- Use the [built-in components](built-in-components.md) catalog to choose stock
  tasks and processors and see their important metadata inputs.
- Read [configuration](configuration.md) for schema v2 provider targets,
  selection, location, concurrency, and metadata concepts.
- Use the [provider reference](providers.md) and
  [provider profiles](provider-profiles.md) for provider-specific modes,
  authentication, endpoints, and examples.
- Read [selectors and regions](selectors-and-regions.md) for `include`,
  `exclude`, AWS account keywords, `all`, and region globs.
- Read [task workflows](task-workflows.md) for dependencies, result sharing,
  recovery data, and scope-aware fan-out/fan-in.
- Read [extension best practices](extension-best-practices.md) to build
  providers, universal tasks, provider tasks, and processors.
- Read [examples](examples.md) for the repository template, GitHub Actions, and
  standalone multi-account script template.
