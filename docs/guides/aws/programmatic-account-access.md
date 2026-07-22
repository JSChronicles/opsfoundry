# AWS Programmatic Account Access

This guide describes two AWS governance access patterns for automation when you need to inspect or manage accounts across an AWS Organization:

- `OrganizationAccountAccessRole` for administrative access from the management account into member accounts.
- `SecurityAccessRole` for delegated administrator access to security/audit data across payer and member accounts. You can create different delegated roles for different groups or people when they need separate access boundaries. This avoids giving routine users or automation access to the payer or management account.

These patterns are useful for Anvil account inventory and task execution, and they are also useful for any programmatic workflow that needs predictable cross-account role assumptions.

## Access Model

Use separate roles for separate governance jobs:

| Pattern | Primary IAM role | Typical permissions | Use when |
| --- | --- | --- | --- |
| Organization access | `OrganizationAccountAccessRole` | `AdministratorAccess` | The management account, or automation running from a trusted management-account role, needs full administrative access to member accounts. |
| Delegated security access | `SecurityAccessRole` | `SecurityAudit` | A delegated administrator account needs read-only security visibility across the organization without using the management account for day-to-day access. |

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

Do not deploy `member-account-org-role-stackset.yaml` to the management account. It is designed for member accounts because it grants the management account authority to assume into the target account.

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

Delegated administrator access should use a separate role from the full administrative organization access role. The provided template defines a `SecurityAccessRole` pattern where an SSO principal in the delegated security account can assume read-only security roles in payer and member accounts.

Use this template for delegated admin security access:

| Template | Deploy from | Deploy to | Purpose |
| --- | --- | --- | --- |
| [`member-account-SecurityAccessRole.yml`](cloudformation/aws_delegated_admin_access/member-account-SecurityAccessRole.yml) | Management account | Payer and member accounts | Creates the target role with `SecurityAudit` permissions and trusts the delegated security account or its selected IAM Identity Center permission set. |

### Caller Permissions

The target account trust policy is only one side of delegated access. The IAM Identity Center permission set used in the delegated security account also needs permission to call `sts:AssumeRole` against the target `SecurityAccessRole`.

For Anvil organization mode, the permission set also needs permission to discover organization accounts and enabled Regions. Add a policy like this to the permission set used by people or automation running Anvil:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DiscoverOrganizationAccounts",
      "Effect": "Allow",
      "Action": [
        "organizations:DescribeOrganization",
        "organizations:ListAccounts"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ListAccountRegions",
      "Effect": "Allow",
      "Action": "account:ListRegions",
      "Resource": "*"
    },
    {
      "Sid": "AllowAssumeSecurityAccessRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::*:role/SecurityAccessRole"
    }
  ]
}
```

The account must also be registered as a delegated administrator for the AWS services whose organization data it needs to access.

Common places to add this example permission include:

- AWS IAM Identity Center permission sets used by security or platform teams.
- IAM roles used by CI/CD, Anvil, inventory collectors, or reporting jobs.

Keep the resource scope aligned to the approved target role name. If you create different delegated roles for different teams, scope each caller policy to only the role names that group should assume.


### Console Deployment Steps

Use this sequence to configure the delegated security account and roll out the target role across the organization. Anvil uses the SSO session directly when processing the delegated security account. It assumes `SecurityAccessRole` only in other selected accounts, including the management account.

#### Prerequisites

1. Confirm that the security account is registered as a delegated administrator for each AWS service used by the security workflow. I personally used cloudformation as the delegated service.
1. Record the 12-digit AWS account ID of the delegated security account.
1. Identify the IAM Identity Center permission set used to run Anvil in the delegated security account.
1. Add the caller policy from the previous section to that permission set.
1. Confirm that trusted access between AWS Organizations and CloudFormation StackSets is enabled when using service-managed StackSet permissions.

#### Create the Payer Target Role

1. Sign in to the AWS Organizations management account, which is sometimes called the payer account.
1. Open CloudFormation in `us-east-1`.
1. Create a regular stack from [`member-account-SecurityAccessRole.yml`](cloudformation/aws_delegated_admin_access/member-account-SecurityAccessRole.yml).
1. Enter the delegated security account ID for `DelegatedAdminAccountId`.
1. Enter the IAM Identity Center permission set name in `PermissionSetName` to restrict access to the intended SSO role.
1. Review and create the stack, including the acknowledgement that the template creates IAM resources with a custom name.

CloudFormation StackSets does not deploy stack instances to the management account, even when the root is selected. The regular stack in this section is therefore required when the security workflow needs access to the management account.

#### Deploy to Member Accounts with StackSets

1. While signed in to the management account, open AWS Organizations and copy the root ID. A root ID starts with `r-`.
1. Open CloudFormation in `us-east-1`, then open StackSets.
1. Create a StackSet from [`member-account-SecurityAccessRole.yml`](cloudformation/aws_delegated_admin_access/member-account-SecurityAccessRole.yml).
1. Choose service-managed permissions, enter the delegated security account ID for `DelegatedAdminAccountId`, and enter the same IAM Identity Center permission set name in `PermissionSetName`.
1. Choose **Deploy to organizational units (OUs)** and enter the root ID.
1. Set the account filter type to **Difference** and enter the delegated security account ID. This deploys to member accounts beneath the root while excluding the delegated security account, where Anvil uses the base SSO session directly.
1. Select `us-east-1` as the deployment Region.
1. Choose conservative concurrency and failure-tolerance settings for the first deployment, then create the stack instances.
1. Wait for the operation to finish and confirm that every intended account reports `CURRENT`.

#### Verify Access

1. Authenticate through the intended IAM Identity Center permission set in the delegated security account.
1. Confirm that organization account discovery and Region discovery succeed.
1. Assume `arn:aws:iam::<target-account-id>:role/SecurityAccessRole` in one test member account.
1. Run `aws sts get-caller-identity` with the assumed credentials and confirm that the returned ARN names `SecurityAccessRole` in the target account.


## Operational Guidance

- Prefer short role sessions unless a workflow has a documented reason to need longer access.
- Keep role names consistent across accounts so automation can derive role ARNs from account IDs.
- Use StackSets for broad member-account rollout and a single-account stack for the management-account target role.
- Run drift detection after importing existing roles or after manual IAM changes.
- Avoid using the management account for recurring security collection when a delegated administrator account can do the work.
- Scope upstream automation roles to `sts:AssumeRole` only for the approved target role names.
- Treat `OrganizationAccountAccessRole` as high-risk because the provided templates attach `AdministratorAccess`.

## Related AWS Documentation

- [Accessing member accounts in an organization with AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html)
- [CloudFormation StackSet deployment targets](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_DeploymentTargets.html)
- [Import AWS resources into a CloudFormation stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/import-resources.html)
- [Using AWS Organizations with other AWS services](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_services.html)
