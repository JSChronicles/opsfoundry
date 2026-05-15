# Python Best Practices

Use these practices for Python code that needs to be maintainable, reviewable, and predictable in automation.

Prefer simple code first. Add abstractions, dependencies, concurrency, and caching only when they solve a real problem.

## Project Layout

Keep modules focused and cohesive. Reusable behavior should live in functions or classes, not in top-level scripts.

Avoid import-time side effects:

- Do not make network calls during import.
- Do not mutate the filesystem during import.
- Do not start expensive work during import.
- Do not require credentials just to import a module.

Command-line entry points should stay thin: parse arguments, call typed project logic, and exit cleanly.

## Dependencies

Prefer the Python standard library unless an external package is clearly justified.

Add a dependency when it meaningfully reduces maintenance burden, improves correctness, or matches an established project pattern. Do not add a package for small wrappers around standard-library behavior.

When a dependency is added, update `pyproject.toml`, refresh `uv.lock`, and make sure the dependency is covered by the project's normal update process.

## Type Hints

Use native type hints for public interfaces and structured data:

```python
def build_index(items: list[str]) -> dict[str, int]:
    return {item: index for index, item in enumerate(items)}
```

Type these consistently:

- Public functions and methods.
- Dataclass fields.
- Class attributes.
- Return values that are not obvious.

Let local variables infer naturally unless an explicit type improves clarity.

## Docstrings

Use docstrings for public functions, classes, and non-obvious behavior.

Google-style docstrings are a good default:

```python
def load_config(path: str) -> dict[str, object]:
    """Load configuration from a JSON file.

    Args:
        path: Path to the configuration file.

    Returns:
        The parsed configuration data.

    Raises:
        FileNotFoundError: If the file does not exist.
        ValueError: If the file content is invalid.
    """
```

Do not write docstrings that merely repeat the function name. Use them to explain inputs, outputs, raised errors, assumptions, and behavior that a reader cannot see from the signature.

## Errors and Logging

Catch specific exceptions when possible. Do not silently swallow errors.

Only catch an exception when the code adds value, such as:

- Adding useful context.
- Logging an operational failure.
- Cleaning up a resource.
- Converting an external error into a project-specific error.
- Recovering in an intentional and documented way.

Use module-level __LOGGER__:

```python
import logging

__LOGGER__ = logging.getLogger(__name__)
```

When catching a generic exception at a boundary, log it with context and re-raise it:

```python
try:
    run_task()
except Exception as error:
    __LOGGER__.exception(f"Unhandled error during task execution: {error}")
    raise
```

Use `raise` without arguments to preserve the original traceback.

## Data Modeling

Use `@dataclass` for stable structured values:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class AccountResult:
    """Result of processing an account in a region."""

    account_id: str
    region: str
    success: bool
```

Use dictionaries for dynamic external data, JSON-like payloads, and schema-less mappings.

Prefer `frozen=True` when values should not change after creation. Consider `slots=True` for large numbers of instances when it improves memory behavior without making the model harder to use.

## Names

Use names that explain intent:

```python
project_name = project_data["project"]["name"]
items = data["items"]
index = 0
```

Avoid single-character names except for true mathematical expressions.

Avoid pass-through aliases for short dictionary access. A variable should add meaning, shorten a complex expression, capture a point-in-time value, or avoid repeating expensive work.

## CLI Code

Use `argparse` when a script accepts options, flags, or multiple inputs:

```python
import argparse

parser = argparse.ArgumentParser(description="Run the project task.")
parser.add_argument("--config", required=True, help="Path to the config file.")
arguments = parser.parse_args()
```

Keep parsing separate from execution. The main function should pass parsed values into typed logic that can be tested without invoking the CLI.

## Concurrency

Do not add threading, async execution, or process pools by default.

Use concurrency when the work is independent, bounded, and meaningfully faster in parallel. Make ordering, error behavior, cancellation, and limits intentional.

Prefer the simplest correct implementation:

```python
for account_id in account_ids:
    process_account(account_id)
```

Add bounded concurrency only after the simple version is too slow for the expected workload.

## Caching

Add caching when it is easy to reason about and provides clear value.

Use standard-library caching for pure, hashable, process-local results:

```python
from functools import lru_cache


@lru_cache(maxsize=256)
def get_account_name(account_id: str) -> str:
    """Return the account name for the given account ID."""
    return load_account_details(account_id)["name"]
```

Before adding a cache, decide how it handles memory growth, stale data, thread-safety, and invalidation.

## Review Checklist

Before finishing Python changes:

- Code has no unsafe import-time side effects.
- Public interfaces and structured objects have native type hints.
- Public functions and classes have useful docstrings.
- Exceptions are specific, visible, and re-raised when appropriate.
- Logs include useful context.
- External dependencies are justified.
- CLIs parse arguments with a maintainable interface.
- Dataclasses are used where they reduce boilerplate.
- Concurrency and caching are justified by clear benefit.
- Variable names explain intent.
- Relevant formatter, linter, type checker, and tests have been run when available.
