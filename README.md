# Ops Foundry

This repository contains the documentation site source for
[opsfoundry.dev](https://opsfoundry.dev).

Ops Foundry will become a centralized documentation hub for DevOps, cloud
automation, infrastructure tooling, reusable engineering patterns, and project
documentation.

The first project documented here will be Anvil. Future documentation may
include Foundry, Forge, guides, templates, and engineering notes.

This is currently a Phase 0 MkDocs site with a simple landing page. Full
project documentation will be added later.

## Local preview

Install the documentation dependencies and run MkDocs with `uv`:

```sh
uv sync --locked
uv run mkdocs serve
```

The local site will be available at <http://127.0.0.1:8000/>.

## GitHub Pages

This repository deploys with GitHub Actions. GitHub Pages should be configured
with **Source: GitHub Actions** and the custom domain `opsfoundry.dev`.

## License

This repository uses a license split by material type:

- Documentation and written content are licensed under Creative Commons
  Attribution 4.0 International
  ([LICENSE-DOCS](LICENSE-DOCS) or https://creativecommons.org/licenses/by/4.0/).
- Code, examples, workflows, configuration, and site assets are licensed under
  the Apache License 2.0
  ([LICENSE-CODE](LICENSE-CODE) or https://www.apache.org/licenses/LICENSE-2.0).

See [LICENSE](LICENSE) for the short license summary.
