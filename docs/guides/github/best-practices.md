# GitHub Best Practices

Use these conventions to keep repositories predictable, searchable, and easy to automate. They are intentionally lightweight: pick names that explain the work, keep changes focused, and use repository templates when a project needs a repeatable setup.

## Naming

Keep names simple and consistent.

| Area | Preferred Pattern | Example |
| --- | --- | --- |
| Repository | `lowercase-words` | `terraform-state-manager` |
| Branch | `<type>/<short-description>` | `feature/add-oidc-role` |
| Pull request | `<type>: <plain-language summary>` | `docs: Add AWS OIDC guide` |
| Commit | Imperative sentence | `Add AWS OIDC setup guide` |

Use these type prefixes as a starting point:

- `feature` or `feat` for new functionality.
- `fix` for bug fixes.
- `docs` for documentation.
- `test` for tests.
- `refactor` for code changes that do not change behavior.
- `chore` for maintenance.
- `release` for release preparation.

### Repository Names

Repository names should be lowercase, descriptive, and separated with hyphens.

Good examples:

- `github-policy-manager`
- `aws-account-inventory`
- `terraform-template-forge`

Avoid:

- Spaces, underscores, and special characters.
- Version names such as `my-service-v2`, unless the version is a truly separate product.
- Environment names such as `website-dev`, unless the environment must be a separate repository.
- Vague names such as `scripts`, `misc`, or `new-tool`.

### Branch Names

Use a short prefix and a short description:


- `feature/add-login-page`
- `fix/refresh-token-expiry`
- `docs/update-readme`
- `chore/update-dependencies`

If your team uses issue IDs, include the ID after the prefix:


- `feature/PROJ-123-add-login-page`


Keep branch names readable in terminals and automation. Use letters, numbers, slashes, and hyphens.

### Pull Request Titles

Use PR titles that explain the change at review time:

- `docs: Add AWS OIDC setup guide`
- `fix: Handle expired OAuth tokens`
- `chore: Update GitHub Actions dependencies`

Keep the title short. Put context, screenshots, migration notes, and test details in the PR description.

## Commits

Commits should make the project history useful.

- Write the subject line in imperative mood: `Add validation`, not `Added validation`.
- Keep the subject short and specific.
- Use the commit body to explain why a change was needed when the reason is not obvious.
- Reference related issues or pull requests when useful.
- Keep commits focused on one logical change.
- Test before committing when the change affects behavior.
- Do not commit generated files, dependencies, build output, or local machine files unless the repository explicitly tracks them.

Good commit message:

```text
Add validation for login email input

Reject malformed email addresses before submitting the login form so users get
immediate feedback and the API receives cleaner requests.
```

bad commit message:

```text
Fix stuff
```

## Template Repository

Create repository templates when a new project should start with a known structure, workflows, linting, tests, and documentation.

Templates are most useful for repeated project types such as:

- Python packages or tools.
- PowerShell modules or scripts.
- Terraform modules or infrastructure repositories.
- GitHub Actions workflow repositories.

A good template should include:

- A README with setup, usage, and maintenance notes.
- A license and ownership information.
- Default linting and testing commands.
- Starter GitHub Actions workflows.
- Dependabot configuration when dependencies are present.
- CODEOWNERS when review ownership matters.

For dependency automation conventions, use [Dependabot Best Practices](../dependabot-best-practices.md).

## Miscellaneous Repositories

A miscellaneous repository is acceptable for small scripts, one-off tools, and snippets that do not yet justify their own repository.

Use a miscellaneous repository when:

- The item is small and rarely changed.
- The script or snippet is useful but does not need a full release process.
- The item does not fit an existing project repository.

Create a dedicated repository when:

- The tool has its own lifecycle, tests, or deployment workflow.
- Multiple people contribute to it regularly.
- The code needs repository-specific permissions, secrets, or GitHub Actions.
- The folder has grown enough that users need separate documentation to understand it.

Keep each tool or script in its own folder:

```text
misc-scripts/
|-- README.md
|-- one-off-report/
|   |-- README.md
|   `-- generate-report.py
|-- maintenance-task/
|   |-- README.md
|   `-- run-maintenance.ps1
`-- helper-tools/
    |-- README.md
    |-- format-data.py
    `-- check-status.sh
```

Each folder should include a short README when the script needs parameters, credentials, environment setup, or examples.

## Automation Fit

These conventions are intentionally compatible with repository automation:

- Dependabot labels and commit prefixes are easier to filter when names are consistent.
- Branch prefixes make rulesets and workflow conditions easier to read.
- PR title prefixes can feed changelogs or release notes when a repository adopts that process.
- Template repositories reduce repeated setup mistakes.

Keep the conventions simple enough that contributors can follow them without needing a separate decision tree.
