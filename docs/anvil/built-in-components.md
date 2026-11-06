# Built-in Components

Anvil ships universal tasks, provider-specific tasks, and result processors.
The installed catalog is authoritative for the environment:

```console
anvil list --tasks
anvil list --processors
anvil validate --tasks --processors
```

Use `anvil list --tasks <name> --detail` for a unique task name or
`anvil list --processors <name> --detail` for generated documentation from the
component's current `run()` docstring. Some task names intentionally occur in
multiple provider packages; configuration resolves them in the selected
provider's catalog.

## Universal Tasks

| Task | Purpose |
| --- | --- |
| `noop` | Successful smoke-test task |
| `noop_fail` | Intentional failure for error-path and reporting tests |

Universal tasks use the provider-neutral task contract and can run with any
provider that supports their declared scope.

## AWS Tasks

| Task | Scope | Purpose and important metadata |
| --- | --- | --- |
| `compare_asg_to_cluster_instances` | `region` | Compare ECS container instances with `<cluster>-asg`; requires `clusters`, optionally `ecs_region` |
| `count_subnets_with_timings` | `region` | Count subnets and expose API timing/retry data |
| `count_vpc` | `region` | Count and list VPC IDs in the current Region |
| `detect_deprecated_lambda_runtimes` | `region` | Return SARIF-compatible findings; requires `runtimes` |
| `get_aws_inline_policies` | `target` | Collect IAM user, role, and group inline policies; optional `types` (`user`, `role`, `group`) |
| `get_aws_sso_inline_policies` | `configured_target` | Collect IAM Identity Center permission-set inline policies; optional `identity_center_region` |
| `get_organization_structure` | `configured_target` | Collect OUs, accounts, SCPs, and Control Tower controls from the management account |
| `list_lambdas_by_runtime` | `region` | List Lambda functions matching required `runtimes` |
| `remove_cloudformation_stack` | `target` | Remove exact stacks selected per account with optional retention of stuck resources; honors dry run |
| `remove_iam_role` | `target` | Remove required IAM `roles` after cleaning attached resources; honors dry run |
| `remove_iam_user` | `target` | Remove required IAM `users` after cleaning their attached resources; honors dry run |
| `remove_idc_user` | `configured_target` | Remove selected Identity Center users; accepts `users`, `status`, and optional `identity_center_region` |
| `remove_missing_group_assignments` | `configured_target` | Remove assignments for missing Identity Center groups; optional `identity_center_region` |

See [Task Scopes](task-scopes.md) for why account-wide and shared-control-plane
tasks must not use the default region scope.

## Azure and GCP Tasks

| Provider | Task | Purpose |
| --- | --- | --- |
| Azure | `count_resource_groups` | Count subscription resource groups; target-scoped and not location-filtered |
| GCP | `get_project_info` | Return Resource Manager metadata for the current project |

## Cloudflare Tasks

| Task | Required target | Purpose and important metadata |
| --- | --- | --- |
| `list_account_member` | Account | List members; optional `members` and `max_results` |
| `list_zone` | Account | List visible zones; optional `max_results` |
| `list_dns_record` | Zone | List DNS records; optional `max_results` |
| `remove_account_member` | Account | Remove required `members`; honors dry run and preserves partial failure results |

## Datadog Tasks

| Task | Purpose and important metadata |
| --- | --- |
| `list_dashboard` | List dashboards; optional `max_results` |
| `list_monitor` | List monitors; optional `max_results` |
| `list_user` | List users; optional `users` and `max_results` |
| `disable_user` | Disable required `users`; honors dry run and preserves partial failure results |

## GitHub Tasks

Repository audit and alert tasks require repository targets unless noted.
Organization membership tasks require organization targets.

| Task | Target and purpose |
| --- | --- |
| `audit_branch_protection` | Repository branch protection; optional `branch` |
| `audit_repo_security_settings` | Repository security and merge settings |
| `audit_rulesets` | Repository rulesets; optional result limit and parent inclusion |
| `list_code_scanning_alert` | Organization or repository code-scanning alerts with supported API filters |
| `list_dependabot_alert` | Organization or repository Dependabot alerts with supported API filters |
| `list_secret_scanning_alert` | Organization or repository secret-scanning alerts with supported API filters |
| `search_code` | Organization or repository code search; requires `query`, optionally `max_results` and `highlight` |
| `list_member` | Organization members; optional `members` and `max_results` |
| `list_team` | Organization teams; optional `teams` and `max_results` |
| `list_team_member` | Members of required `teams`; optional `members` and `max_results` |
| `remove_member` | Remove required organization `members`; honors dry run |
| `remove_team` | Delete required organization `teams`; honors dry run |
| `remove_team_member` | Remove required `members` from required `teams`; honors dry run |

## GitLab Tasks

| Task | Target and purpose |
| --- | --- |
| `audit_branch_protection` | Audit project protected branches |
| `audit_repo_security_settings` | Audit project security settings |
| `audit_rulesets` | Audit project push rules and approval configuration |
| `list_code_scanning_alert` | List project SAST findings |
| `list_dependabot_alert` | List project dependency findings |
| `list_secret_scanning_alert` | List project secret-detection findings |
| `search_code` | Search project blobs; requires `query`, optionally `max_results` |
| `list_member` | List group or project members; optional `members` and `max_results` |
| `remove_member` | Remove required member IDs from a group or project; honors dry run |

## PagerDuty Tasks

| Task | Purpose and important metadata |
| --- | --- |
| `list_escalation_policy` | List escalation policies; optional `max_results` |
| `list_service` | List services; optional `max_results` |
| `list_team` | List teams; optional `max_results` |
| `list_user` | List users; optional `users` and `max_results` |
| `remove_user` | Remove required user IDs in `users`; honors dry run and preserves partial failure results |

## Built-in Processors

| Processor | Use |
| --- | --- |
| `html_report` | Produce a self-contained human-readable report from completed results |
| `sarif_report` | Produce SARIF 2.1.0 from `detect_` task payloads containing `sarif_findings` |

Processors can run from target `post_run` configuration or against an existing
run with `anvil results --results-dir ... --processor ...`. See the
[CLI reference](cli.md#processors) and
[extension best practices](extension-best-practices.md#build-a-processor).
