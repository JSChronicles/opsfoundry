# Dependabot Best Practices

This guide describes a conservative Dependabot setup for repositories that want predictable dependency updates, readable pull requests, and clean automation labels.

## Configuration Goals

Use Dependabot configuration to make updates easy to review:

- Track each package ecosystem explicitly.
- Use a predictable schedule and timezone.
- Add a cooldown window so brand-new releases have time to settle.
- Group routine version updates by ecosystem.
- Use commit message prefixes that identify the ecosystem.
- Apply labels that make filtering and ruleset automation straightforward.

## Naming Conventions

Use names that make the dependency source obvious in commit history, pull request titles, labels, and groups.

| Area | Pattern | Example |
| --- | --- | --- |
| Group name | Ecosystem or manifest family | `uv-dependencies` |
| Commit prefix | `chore(<ecosystem>)` | `chore(uv)` |
| Labels | `dependencies` plus ecosystem labels | `dependencies`, `python`, `uv` |

Prefer stable names over clever ones. Names like `uv-dependencies`, `github-actions`, and `pre-commit` are easy to search and map directly to the package ecosystem that created the pull request.

## Package Ecosystems

Define each ecosystem separately, even when all manifests live in the repository root. This keeps schedules, labels, groups, and future exceptions isolated.

Common ecosystem entries for this repo style:

- `uv` for Python dependencies managed by `uv`.
- `github-actions` for workflow action pins.
- `pre-commit` for hooks in `.pre-commit-config.yaml`.

## Schedule and Cooldown

Monthly updates are a good default for small repositories because they reduce pull request noise while still keeping dependencies fresh.

Use a fixed local timezone so Dependabot activity happens during normal review hours:

```yaml
schedule:
  interval: "monthly"
  time: "08:00"
  timezone: "America/Chicago"
```

Use `cooldown.default-days` to delay routine version update pull requests. A 7 day cooldown gives maintainers and upstream projects time to catch broken releases, bad tags, or fast follow-up patches before Dependabot opens a pull request. 14 days is more conservative, but the practical benefit usually tapers off after that.

```yaml
cooldown:
  default-days: 7
```

When a repository tracks multiple ecosystems, offset the scheduled times for each ecosystem instead of starting them all at the same minute. This reduces branch churn, out-of-date pull requests, and race conditions when Dependabot opens several updates against the same base branch.

For example:

```yaml
- package-ecosystem: "uv"
  directory: "/"
  schedule:
    interval: "monthly"
    time: "08:00"
    timezone: "America/Chicago"

- package-ecosystem: "github-actions"
  directory: "/"
  schedule:
    interval: "monthly"
    time: "08:30"
    timezone: "America/Chicago"

- package-ecosystem: "pre-commit"
  directory: "/"
  schedule:
    interval: "monthly"
    time: "09:00"
    timezone: "America/Chicago"
```

## Grouping

Group version updates within each ecosystem to reduce pull request volume:

```yaml
groups:
  uv-dependencies:
    applies-to: version-updates
    patterns:
      - "*"
```

Keep the group scoped to one ecosystem. A single pull request that updates Python packages, workflow actions, and pre-commit hooks is harder to review and harder to roll back.

For Terraform and OpenTofu repositories, keep the same grouping pattern per ecosystem:

```yaml
- package-ecosystem: "terraform"
  directory: "/terraform"
  schedule:
    interval: "monthly"
    time: "08:00"
    timezone: "America/Chicago"
  cooldown:
    default-days: 7
  groups:
    terraform:
      applies-to: version-updates
      patterns:
        - "*"
  commit-message:
    prefix: "chore(terraform)"
  labels:
    - "dependencies"
    - "terraform"
```

Use `directory` when one path owns the ecosystem. Use `directories` when the repository has multiple Terraform module paths and Dependabot should check each of them explicitly:

```yaml
- package-ecosystem: "terraform"
  directories:
    - "/terraform"
    - "/terraform/network"
    - "/terraform/security"
    - "/terraform/observability"
  schedule:
    interval: "monthly"
  cooldown:
    default-days: 7
```

This keeps routine Terraform updates grouped while still making it clear which module paths are part of Dependabot coverage.

## Example Configuration

Add this file at `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "uv"
    directory: "/"
    schedule:
      interval: "monthly"
      time: "08:00"
      timezone: "America/Chicago"
    cooldown:
      default-days: 7
    groups:
      uv-dependencies:
        applies-to: version-updates
        patterns:
          - "*"
    commit-message:
      prefix: "chore(uv)"
    labels:
      - "dependencies"
      - "python"
      - "uv"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
      time: "08:30"
      timezone: "America/Chicago"
    cooldown:
      default-days: 7
    groups:
      github-actions:
        applies-to: version-updates
        patterns:
          - "*"
    commit-message:
      prefix: "chore(github-actions)"
    labels:
      - "dependencies"
      - "github-actions"

  - package-ecosystem: "pre-commit"
    directory: "/"
    schedule:
      interval: "monthly"
      time: "09:00"
      timezone: "America/Chicago"
    cooldown:
      default-days: 7
    groups:
      pre-commit:
        applies-to: version-updates
        patterns:
          - "*"
    commit-message:
      prefix: "chore(pre-commit)"
    labels:
      - "dependencies"
      - "pre-commit"
```

## Pairing With Auto-Approval

If the repository also uses [Dependabot Auto-Approval Setup](dependabot-auto-approval.md), keep auto-approval limited to patch and minor version updates.

Do not auto-approve major updates by default. Major updates should stay visible for manual review because they are more likely to include breaking changes, migration work, or policy decisions.

The [single-contributor](dependabot-auto-approval.md#personal-repository-single-contributor) and [GitHub App](dependabot-auto-approval.md#repository-with-multiple-contributors) auto-approval workflows also work with repositories that have multiple Dependabot ecosystems. Use the [filtered auto-update workflow](dependabot-auto-approval.md#filtered-auto-update-workflow) only when specific ecosystems, update types, or dependency name prefixes should be eligible for automatic approval and auto-merge.
