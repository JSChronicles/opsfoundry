# AWS OIDC for GitHub Actions

This guide describes the AWS setup needed for GitHub Actions to authenticate with AWS by using OpenID Connect (OIDC), without storing long-lived AWS access keys in GitHub secrets.

Use this setup when a repository needs to deploy infrastructure, read organization data, manage Terraform state, or assume roles across AWS accounts.

## Assumptions

- The AWS Organizations management or payer account is the account that owns the GitHub Actions OIDC provider.
- Member accounts have the standard `OrganizationAccountAccessRole`, unless your organization uses an equivalent custom role.
- Terraform state is stored in an S3 bucket with a known prefix.
- GitHub Actions workflows use repository-scoped access, not broad organization-wide trust.

## OrganizationAccountAccessRole

AWS Organizations creates `OrganizationAccountAccessRole` in member accounts when those accounts are provisioned through the organization.

This guide assumes that role exists and has not been modified in accounts where cross-account administrative access is required. If your organization restricts or replaces this role, use an equivalent role with the required permissions and update the trust policy and permissions examples below.

## Create the OIDC Provider

In the AWS management or payer account:

1. Open IAM. > **Identity providers** > **Add provider** > Select **OpenID Connect**
    1. Set **Provider URL** to `https://token.actions.githubusercontent.com`.
    1. Set **Audience** to `sts.amazonaws.com`.
1. Create the provider.

## Create the GitHub Actions Role

Create an IAM role that GitHub Actions can assume.

1. Open IAM > **Roles** > Choose **Create role** > Select **Custom trust policy**
    1. Add a trust policy like this, replacing the placeholder values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::{payer_account_id}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:{github_org}/{github_repository}:*"
        }
      }
    }
  ]
}
```

Scope the `githubusercontent.com:sub` condition as tightly as possible. Prefer a single repository, and only widen it when multiple repositories genuinely need the same role.

## Add Permissions

Attach only the managed policies and inline policies the workflow needs. For organization discovery or infrastructure workflows, common permissions include:

- Read-only AWS Organizations access.
- Permission to assume approved cross-account roles.
- Permission to read and write the Terraform state bucket path used by the repository.

Use an inline policy like this when the workflow must assume roles in member accounts:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AssumeApprovedRoles",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::{log_archive_account_id}:role/AWSControlTowerExecution",
        "arn:aws:iam::*:role/OrganizationAccountAccessRole"
      ]
    }
  ]
}
```

Use an inline policy like this for Terraform state access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::{terraform_state_bucket}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::{terraform_state_bucket}/{path}",
        "arn:aws:s3:::{terraform_state_bucket}/{path}/*"
      ]
    }
  ]
}
```

## Workflow Usage

In GitHub Actions, grant the workflow permission to request an OIDC token:

```yaml
permissions:
  contents: read
  id-token: write
```

Then configure AWS credentials by assuming the role:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@d979d5b3a71173a29b74b5b88418bfda9437d885 # v6.1.1
  with:
    role-to-assume: arn:aws:iam::{payer_account_id}:role/{github_actions_role_name}
    aws-region: us-east-1
```

## Review Checklist

Before using the role in production:

- Confirm the trust policy is scoped to the expected GitHub organization and repository.
- Confirm the workflow has `id-token: write`.
- Confirm the role does not allow broader AWS access than the workflow needs.
- Confirm Terraform state access is limited to the expected bucket and prefix.
- Confirm cross-account assume-role permissions only include approved roles.
- Keep workflow names, branches, and pull requests consistent with the local [GitHub best practices](github-best-practices.md).
