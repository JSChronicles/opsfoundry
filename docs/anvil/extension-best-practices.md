# Extension Best Practices

Anvil extensions can contribute providers, universal tasks, provider-specific
tasks, and post-run processors. Components are discovered from registered
package roots; each public module or immediate provider child is one component.

## Choose the Smallest Extension Boundary

- Create a **universal task** when the operation uses only provider-neutral
  context or deliberately branches by `provider`.
- Create a **provider task** when the operation relies on one provider's
  session, target type, API, or resource semantics.
- Create a **processor** when the operation transforms or exports completed run
  results rather than calling a provider during task execution.
- Create a **provider** only when Anvil needs a new authentication, discovery,
  execution-target, location, or session model.

Keeping these boundaries narrow makes validation, installation, and failure
handling easier to reason about.

## Package Layout

A distribution can expose any combination of component packages:

```text
company_anvil/
  universal_tasks/
    __init__.py
    normalize_inventory.py
  aws_tasks/
    __init__.py
    enforce_bucket_policy.py
  processors/
    __init__.py
    inventory_csv.py
  providers/
    examplecloud/
      __init__.py
      provider.py
      session.py
```

Register package roots, not individual components:

```toml
[project.entry-points."anvil.providers.tasks"]
company-universal = "company_anvil.universal_tasks"

[project.entry-points."anvil.providers.aws.tasks"]
company-aws = "company_anvil.aws_tasks"

[project.entry-points."anvil.processors"]
company-processors = "company_anvil.processors"

[project.entry-points."anvil.provider_packages"]
company-providers = "company_anvil.providers"
```

For another provider task package, replace `aws` with the provider's current
component name. Duplicate applicable component names are rejected as ambiguous.

## Build a Universal Task

Every task module exposes a keyword-only `run()` function:

```python
from anvil.actions import ActionRecorder


def run(
    *,
    provider: str,
    execution_target_id: str,
    execution_target_name: str,
    execution_target_type: str,
    region: str,
    session,
    dry_run: bool,
    metadata: dict[str, object],
    dependency_data: dict[str, object],
    actions: ActionRecorder,
) -> dict[str, object]:
    """Validate provider-neutral workflow inputs."""

    required_labels = metadata.get("required_labels", [])
    if not isinstance(required_labels, list):
        raise RuntimeError("metadata.required_labels must be an array")

    actions.record("Validated required label configuration")
    return {"valid": True, "required_labels": required_labels}
```

Universal task best practices:

- Do not assume a provider SDK or session interface unless branching explicitly
  on `provider`.
- Return only JSON-serializable values.
- Keep credentials and secret-derived values out of results and actions.
- Use `metadata` for static operator inputs and `dependency_data` for upstream
  task values.
- Add a useful Google-style `run()` docstring; it powers `--detail` output.

## Build a Provider-Specific Task

Provider tasks use the same signature but may rely on the selected provider's
session. Validate provider and target assumptions before API calls:

```python
def run(*, provider: str, execution_target_type: str, session, dry_run: bool,
        metadata: dict[str, object], actions, **kwargs) -> dict[str, object]:
    """List resources from the current ExampleCloud project."""

    if provider != "examplecloud" or execution_target_type != "project":
        raise RuntimeError("list_widget requires an ExampleCloud project target")

    widgets = list(session.list_widgets())
    actions.record(f"Listed {len(widgets)} widget(s)")
    return {"widgets": widgets}
```

Provider task best practices:

- Treat the session as already scoped to the current target and location.
- Use SDK pagination helpers for list operations.
- Validate required metadata types before provider calls.
- Check `dry_run` before every mutation and prefix planned log/action messages
  with `(dry-run)`.
- Let unexpected SDK errors surface unless you can add actionable context.
- Use singular resource names such as `list_user` or `remove_dns_record`; accept
  arrays through plural metadata keys when selecting multiple resources.

Tasks are region-scoped by default. Declare `TASK_SCOPE = "target"` only when
the operation should run once per execution target. Do not configure scope in
YAML. Providers advertise the scopes they support.

## Build a Processor

A processor runs after task results are written. Its module exposes:

```python
from pathlib import Path

from anvil.processor_loader import ProcessorRunContext


def run(
    *,
    context: ProcessorRunContext,
    output: str | None,
    metadata: dict[str, object],
) -> dict[str, object]:
    """Write a custom report from completed Anvil results."""

    output_path = (
        Path(output)
        if output is not None
        else context.run_dir / "reports" / "inventory.txt"
    )
    output_path.parent.mkdir(parents=True, exist_ok=True)
    output_path.write_text(str(context.summary), encoding="utf-8")
    return {"output": str(output_path)}
```

Configure it under `post_run`:

```yaml
post_run:
  - processor: inventory_csv
    output: inventory.csv
    metadata:
      include_headers: true
    run_on_failure: true
```

Processor best practices:

- Use `context.target_result` for one configured target and
  `context.target_results` for the complete available result set.
- Treat a non-null `output` as an already resolved path beneath the run's
  `reports` directory. Choose a safe default beneath `context.run_dir` when it
  is null.
- Treat processor metadata as invocation-local configuration.
- Return a small JSON-serializable summary of written artifacts.
- Use `run_on_failure` only when partial results are useful.
- Add a `run()` docstring for generated detail documentation.

## Build a Provider

Register a collection package through `anvil.provider_packages`. Each immediate
child is a provider package and exposes `create_provider_instance()`:

```python
from company_anvil.providers.examplecloud.provider import ExampleCloudProvider


def create_provider_instance() -> ExampleCloudProvider:
    """Create the ExampleCloud provider adapter."""

    return ExampleCloudProvider()
```

The provider instance defines `ProviderMetadata` and implements the provider
contract:

```python
metadata = ProviderMetadata(
    name="examplecloud",
    display_name="ExampleCloud",
    description="ExampleCloud provider",
    default_regions=("global",),
    supported_task_scopes=frozenset({"region", "target"}),
)
```

Required provider responsibilities are:

1. validate provider-specific target modes, options, selectors, and locations;
2. resolve YAML and CLI target filters;
3. return a secret-safe authentication cache identity;
4. perform an actionable authentication check;
5. discover provider locations;
6. prepare reusable provider state;
7. resolve configured targets into `ExecutionTarget` objects;
8. create a runtime that builds sessions, records region outcomes, and closes
   resources.

Providers that advertise `configured_target` scope must also validate task
configuration and prepare a configured-target runtime.

Provider best practices:

- Keep `__init__.py` and offline discovery paths import-safe; import optional
  SDKs only when the provider is selected.
- Keep secrets out of cache keys, logs, metadata, and exceptions. Hash or
  fingerprint credential identity when caching depends on secret rotation.
- Make auth and dependency errors actionable, including the correct optional
  install extra.
- Put provider-specific state in `provider_data`, not engine globals.
- Reuse sessions only within safe credential, target, thread, and location
  boundaries.
- Close clients and other runtime resources deterministically.
- Resolve selectors to concrete targets and locations before task execution.
- Preserve deterministic target and result ordering under concurrency.

## Testing and Validation

Test the narrowest contract first:

- import safety without optional SDKs;
- provider target and option validation;
- authentication error redaction;
- target and location resolution;
- task dry-run behavior and metadata validation;
- processor output and failure behavior;
- duplicate component-name handling;
- extension entry-point discovery.

Run Anvil's offline validation commands before provider API tests:

```console
anvil validate --providers
anvil validate --tasks
anvil validate --processors
anvil validate --config-file examples/example.yaml
```

Use `anvil list --tasks <name> --detail` and
`anvil list --processors <name> --detail` to review operator-facing docs.

For runtime data flow, see [task workflows](task-workflows.md). For the complete
task callable contract, see [task contract](task-contract.md).
