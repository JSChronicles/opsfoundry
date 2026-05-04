# Examples

This page collects practical examples and reference patterns for running Anvil.

## GitHub Actions

For a complete GitHub Actions example that runs Anvil with AWS OIDC and uploads
generated JSON results as workflow artifacts, see
[examples/github-actions](https://github.com/JSChronicles/anvil/blob/main/examples/github-actions/README.md).

## YAML Examples

For runnable YAML examples covering different target and task patterns, see the
[Anvil examples directory](https://github.com/JSChronicles/anvil/tree/main/examples).

## Result Examples

For task examples that show returned result data and `ActionRecorder` usage, see
the [Anvil Results examples](https://github.com/JSChronicles/anvil/tree/main/examples/Results).

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
