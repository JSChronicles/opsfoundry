# uv and pyproject.toml Best Practices

Use `uv` and `pyproject.toml` together to keep Python projects reproducible, easy to set up, and friendly to automation.

The goal is simple: `pyproject.toml` describes the project, `uv.lock` records the resolved dependency graph, and common commands work the same way locally and in CI.

## Project Metadata

Every Python project should define the standard `[project]` fields:

```toml
[project]
name = "example-project"
version = "0.1.0"
description = "Short description of the project."
readme = "README.md"
requires-python = ">=3.14"
dependencies = []
```

Keep the metadata accurate:

- Use a stable package or project name.
- Set `requires-python` to the oldest Python version the project actually supports (that you have tested)
- Keep the description short and factual.
- Use `README.md` as the main entry point for setup and usage.

For a new project, start with the newest stable Python version that your runtime, tooling, and deployment target support. Do the same for dependencies: prefer current stable package versions instead of starting from old pins. Older versions are useful when maintaining an existing project, matching a platform constraint, or avoiding a known regression, but they should not be the default for new work.

## Application vs Package

Decide whether the repository is an installable package or a project workspace.

For a documentation site, internal automation repo, or script collection that does not build a Python package, use:

```toml
[tool.uv]
package = false
```

For a library, CLI, or package that should be built and installed, keep packaging enabled and define the package layout intentionally.

## Dependencies

Use `dependencies` for runtime requirements:

```toml
[project]
dependencies = [
  "boto3>=1.43.8,<2",
  "pydantic>=2.10,<3",
]
```

Use dependency groups for development-only tools:

```toml
[dependency-groups]
dev = [
  "pytest>=9.0.3,<10",
  "ruff>=0.15.13,<1",
]
```

Keep dependencies direct and explainable. If the project imports a package directly, list it directly. Do not add transitive dependencies unless the project imports or configures them itself.

Use lower bounds to describe the minimum version the project needs. Use upper bounds when a dependency has a known compatibility boundary or when the project should not automatically cross major versions. This is especially useful for libraries, reusable tooling, and infrastructure automation where an unexpected major-version upgrade can break downstream users or CI at the wrong time.

For applications with a committed `uv.lock`, the lockfile already controls the exact installed versions. Upper bounds are still useful when they describe real compatibility intent, but avoid overly narrow pins that make routine updates harder.

## Lockfile

Commit `uv.lock` for applications, tools, documentation sites, and infrastructure automation. The lockfile makes local runs and CI runs resolve the same dependency versions.

Use the locked environment in automation:

```sh
uv sync --locked
uv run pytest
```

Use `uv lock` or `uv sync` when intentionally updating dependencies, then review the lockfile diff before merging.

## Reproducible Updates

Use `exclude-newer` by default so dependency resolution avoids brand-new releases:

```toml
[tool.uv]
exclude-newer = "1 week"
```

This gives new package releases time to settle before they are pulled into local development or CI. It also helps avoid broken releases, bad metadata, compromised packages, and fast follow-up patch churn.

If you also need `package = false`, keep both settings in the same `[tool.uv]` table:

```toml
[tool.uv]
package = false
exclude-newer = "1 week"
```

Do not define `[tool.uv]` more than once in the same file.

When starting a new project, choose dependency versions that are already compatible with the one-week cutoff. If a project must immediately use a package version published inside that window, treat it as an intentional exception and revisit the setting once the version ages past the cutoff.

## Tool Configuration

Keep tool configuration in `pyproject.toml` when the tool supports it. This keeps linting, formatting, testing, and packaging behavior visible in one place.

For code-level conventions such as type hints, docstrings, logging, CLIs, data modeling, concurrency, and caching, use [Python Best Practices](python-best-practices.md).

Example Ruff configuration:

```toml
[tool.ruff]
target-version = "py314"

[tool.ruff.lint]
select = ["E4", "E7", "E9", "F", "N"]
task-tags = ["TODO", "FIXME", "HACK"]

[tool.ruff.format]
skip-magic-trailing-comma = true
```

Match `tool.ruff.target-version` to the project Python version. If `requires-python = ">=3.14"`, use `target-version = "py314"` unless the project intentionally only targets a newer version.

## Common Commands

Document the commands contributors should use:

```sh
uv sync --locked
uv run pytest
uv run ruff check .
uv run ruff format .
```

Use the same commands in CI where practical. This reduces drift between local development and automated checks.

## Dependabot

When using Dependabot with `uv`, keep the dependency update pattern consistent with the repository's automation.

Use the local [Dependabot Best Practices](dependabot-best-practices.md) guide for schedules, grouping, labels, and commit message conventions.

## Review Checklist

Before merging changes to Python project configuration:

- `pyproject.toml` parses correctly.
- `uv.lock` is updated when dependencies change.
- Runtime dependencies and development dependencies are separated.
- `requires-python` matches the versions tested in CI.
- `[tool.uv]` includes `exclude-newer = "1 week"` unless there is a documented temporary exception.
- Tool versions and settings are configured in one predictable place.
- The README includes the expected `uv` setup and test commands.
