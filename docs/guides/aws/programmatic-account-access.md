# AWS Programmatic Account Access

This guide describes two AWS governance access patterns for automation when you need to inspect or manage accounts across an AWS Organization:

- `OrganizationAccountAccessRole` for administrative access from the management account into member accounts.
- `SecurityAccessRole` for delegated administrator access to security/audit data across payer and member accounts. You can create different delegated roles for different groups or people when they need separate access boundaries. This avoids giving routine users or automation access to the payer or management account.

These patterns are useful for Anvil account inventory and task execution, and they are also useful for any programmatic workflow that needs predictable cross-account role assumptions.

## Access Model

Use separate roles for separate governance jobs:

| Pattern | Primary role | Typical permissions | Use when |
| --- | --- | --- | --- |
| Organization access | `OrganizationAccountAccessRole` | `AdministratorAccess` | The management account, or automation running from a trusted management-account role, needs full administrative access to member accounts. |
| Delegated security access | `SecurityAccessRole` | `SecurityAudit` in target accounts | A delegated administrator account needs read-only security visibility across the organization without using the management account for day-to-day access. |

Keep the management account path narrow. Use `OrganizationAccountAccessRole` for bootstrapping and account administration, and prefer delegated administrator accounts for recurring security, audit, and reporting workflows.

## OrganizationAccountAccessRole

AWS Organizations creates `OrganizationAccountAccessRole` automatically when an account is created through AWS Organizations. Invited accounts, legacy accounts, or accounts where the role was manually changed may need the role created or brought under infrastructure-as-code management.

Use these templates for the organization access role:

| Template | Deploy from | Deploy to | Purpose |
| --- | --- | --- | --- |
| [`member-account-org-role-stackset.yaml`](cloudformation/aws_org_role_access/member-account-org-role-stackset.yaml) | Management account | Member accounts through StackSets | Creates `OrganizationAccountAccessRole` in member accounts and trusts the management account root principal. |
| [`management-account-org-role-stack.yaml`](cloudformation/aws_org_role_access/management-account-org-role-stack.yaml) | Management account | Management account only | Creates the management-account copy of `OrganizationAccountAccessRole` with trust for the AWS Organizations service. |
| [`management-account-org-role-stack-import.yaml`](cloudformation/aws_org_role_access/management-account-org-role-stack-import.yaml) | Management account | Management account only | Imports an existing management-account role into CloudFormation management without replacing it. |

### Member Account StackSet

Use [`member-account-org-role-stackset.yaml`](cloudformation/aws_org_role_access/member-account-org-role-stackset.yaml) when member accounts need a consistent `OrganizationAccountAccessRole`.

Deploy it as a CloudFormation StackSet from the management account to the intended organizational units or accounts. The template takes `ManagementAccountId` and creates this trust relationship in each target account.

Use this for:

- Newly invited accounts that do not already have the role.
- Existing accounts where the role was removed.
- Standardizing the role definition across member accounts.
- Making account access predictable for Anvil or other inventory and execution tooling.

Do not deploy this StackSet to the management account. It is designed for member accounts because it grants the management account authority to assume into the target account.

### Management Account Stack

Use [`management-account-org-role-stack.yaml`](cloudformation/aws_org_role_access/management-account-org-role-stack.yaml) only in the AWS Organizations management account.

This template creates a management-account role named `OrganizationAccountAccessRole` and trusts the AWS Organizations service principal.

Use this when the management account role should be created and tracked by CloudFormation from the start. Keep this as a single-account stack, not a StackSet.

This role is different from the member-account access role. Member account roles trust the management account root principal. The management-account stack trusts the AWS Organizations service.

### Import Existing Management Role

Use [`management-account-org-role-stack-import.yaml`](cloudformation/aws_org_role_access/management-account-org-role-stack-import.yaml) only when the management account already has the role and CloudFormation does not manage it yet.

The import template includes `DeletionPolicy: Retain`, which is required for CloudFormation resource imports and also protects the role if the stack is later deleted. Use it for adoption, not for normal creation.

Recommended import flow:

1. Confirm the existing IAM role is named `OrganizationAccountAccessRole`.
1. Confirm the role's trust policy and attached policies match the import template.
1. Start a CloudFormation resource import operation with `management-account-org-role-stack-import.yaml`.
1. Identify the existing resource by role name: `OrganizationAccountAccessRole`.
1. Execute the import change set.
1. Run drift detection after import and resolve any differences before using the stack as the source of truth.

Do not use the import template to create a new role. Use the normal management account stack when the role does not already exist.

## Delegated Admin Security Access

Delegated administrator access should use a separate role from the full administrative organization access role. The provided templates define a `SecurityAccessRole` pattern where a delegated admin account can assume read-only security roles in payer and member accounts.

Use these templates for delegated admin security access:

| Template | Deploy from | Deploy to | Purpose |
| --- | --- | --- | --- |
| [`delegated-admin-account-SecurityAccessRole-stack.yml`](cloudformation/aws_delegated_admin_access/delegated-admin-account-SecurityAccessRole-stack.yml) | Delegated admin account | Delegated admin account only | Creates the source role that can assume `SecurityAccessRole` in other accounts. |
| [`member-account-SecurityAccessRole.yml`](cloudformation/aws_delegated_admin_access/member-account-SecurityAccessRole.yml) | Management account or delegated deployment pipeline | Payer and member accounts | Creates the target role with `SecurityAudit` permissions and trusts the delegated admin account. |

### Delegated Admin Account Role

Deploy [`delegated-admin-account-SecurityAccessRole-stack.yml`](cloudformation/aws_delegated_admin_access/delegated-admin-account-SecurityAccessRole-stack.yml) in the delegated administrator account.

This role is the source role for security automation. It trusts principals in the delegated admin account and grants permission to assume `SecurityAccessRole` in other accounts.

Use this role for workflows that run from the delegated admin account, such as security inventory, audit collection, compliance checks, or Anvil tasks that should not require management-account credentials.

### Caller Permissions

The target account trust policy is only one side of delegated access. The principal that starts the session also needs permission to call `sts:AssumeRole` against the target `SecurityAccessRole`.

Add an allow statement like this to the role, permission set, or automation identity that users or workflows authenticate through:

```json
{
  "Sid": "AllowAssumeSecurityAccessRole",
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::*:role/SecurityAccessRole"
}
```

Common places to add this example permission include:

- AWS IAM Identity Center permission sets used by security or platform teams.
- IAM roles used by CI/CD, Anvil, inventory collectors, or reporting jobs.
- Delegated admin account roles that broker access into member accounts.

Keep the resource scope aligned to the approved target role name. If you create different delegated roles for different teams, scope each caller policy to only the role names that group should assume.

### Member and Payer Target Role

Deploy [`member-account-SecurityAccessRole.yml`](cloudformation/aws_delegated_admin_access/member-account-SecurityAccessRole.yml) to each target account that the delegated admin account must inspect. This includes member accounts and, when required, the payer or management account.

The template creates the target `SecurityAccessRole`, attaches AWS managed `SecurityAudit`, and configures trust for the delegated admin account.

Use StackSets when applying this role broadly across organizational units. Use direct stack deployment for exceptional accounts that need separate rollout control.

## Operational Guidance

- Prefer short role sessions unless a workflow has a documented reason to need longer access.
- Keep role names consistent across accounts so automation can derive role ARNs from account IDs.
- Use StackSets for broad member-account rollout and single-account stacks for management or delegated admin account roles.
- Run drift detection after importing existing roles or after manual IAM changes.
- Avoid using the management account for recurring security collection when a delegated administrator account can do the work.
- Scope upstream automation roles to `sts:AssumeRole` only for the approved target role names.
- Treat `OrganizationAccountAccessRole` as high-risk because the provided templates attach `AdministratorAccess`.

## Related AWS Documentation

- [Accessing member accounts in an organization with AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html)
- [Import AWS resources into a CloudFormation stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/import-resources.html)
- [Using AWS Organizations with other AWS services](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_services.html)
