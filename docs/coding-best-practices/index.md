<div class="product-hero">
  <img src="../images/coding-best-practices.png" alt="Coding Best Practices logo">
  <div>
    <h1>Coding Best Practices</h1>
    <p>Reusable, provider-neutral skills for writing, reviewing, and maintaining code and infrastructure with AI coding agents.</p>
  </div>
</div>

The Coding Best Practices plugin keeps each skill in one canonical repository
and exposes the same guidance to Codex, the ChatGPT desktop app, Claude Code,
and agents that support the Agent Skills layout. Install the complete plugin or
adapt an individual skill at user or project scope.

[View the repository](https://github.com/JSChronicles/coding-best-practices){ .md-button .md-button--primary }
[Report an issue](https://github.com/JSChronicles/coding-best-practices/issues/new){ .md-button }

## Included Skills

<div class="skill-grid">
  <article class="skill-card">
    <h3>Python</h3>
    <p>Modern Python 3.14+ structure, typing, error handling, CLIs, testing, concurrency, and maintainability.</p>
  </article>
  <article class="skill-card">
    <h3>PowerShell</h3>
    <p>PowerShell 7.6+ conventions for advanced functions, validation, errors, logging, testing, and module design.</p>
  </article>
  <article class="skill-card">
    <h3>Terraform</h3>
    <p>Maintainable infrastructure code, safe state and provider usage, validation, modules, testing, and automation.</p>
  </article>
</div>

Each skill is maintained under the repository's `skills/` directory. Product
manifests point to those canonical directories, so a change is made once rather
than copied into separate Codex and Claude packages. TypeScript and Go skills
can be added later using the same structure.

## Install in Codex or the ChatGPT Desktop App

Open the plugin manager and add a marketplace with these values:

| Field | Value |
| --- | --- |
| Source | `JSChronicles/coding-best-practices` |
| Git ref | `main` |
| Sparse paths | Leave blank |

After the marketplace is added, open **Coding Best Practices**, install the
plugin, and start a new chat so the newly installed skills are loaded.

Codex CLI users can add the same marketplace from a terminal:

```console
codex plugin marketplace add JSChronicles/coding-best-practices --ref main
```

Refresh configured marketplaces after publishing an update:

```console
codex plugin marketplace upgrade
```

## Install in Claude Code

Run these commands inside Claude Code:

```text
/plugin marketplace add JSChronicles/coding-best-practices
/plugin install coding-best-practices@coding-best-practices
```

Start a new Claude Code session after installation or an upgrade.

## Use with Other Agents

Agents that support the Agent Skills layout can install individual directories
from `skills/`. The exact installation location and update mechanism depend on
the agent. The skill instructions avoid product-specific language, which keeps
the core guidance portable even when a small host-specific manifest or adapter
is required.

## How the Skills Activate

Installing the plugin makes the skills available; it does not force every skill
into every prompt. The agent selects a skill when the request matches its scope,
such as reviewing Python, authoring a PowerShell module, or checking Terraform.
You can also name the desired skill explicitly in your request.

The repository is currently at version `0.1.0` and is licensed under MIT.
