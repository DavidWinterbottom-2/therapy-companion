# Therapy Companion

Self-hosted Ollama-based daily reflection companion that pushes back, tracks growth focus areas over time, and helps review WhatsApp thread responses.

## Develop

Open in a VS Code **devcontainer** (Reopen in Container) or Codespaces for a
reproducible environment — see [`.devcontainer/`](.devcontainer/).

```bash
pip install -e .[dev]
ruff check . && ruff format --check .          # §10 lint + format
pytest -q --cov --cov-fail-under=80            # §4 tests, 80% coverage floor
```

## Standards

This repo follows [REPO-STANDARDS](https://github.com/DavidWinterbottom-2/devcontainer-sandbox/blob/main/standards/REPO-STANDARDS.md).
The shared skills library and this scaffold's workflows are kept current by
[`template-sync`](.github/workflows/template-sync.yml).

It serves a web UI, so it uses the shared **winterbottom design system** (§8) —
vendor `winterbottom.css` / `winterbottom-theme.js` from `docker-infra/design-system`
(see the `design-system-sync` skill).

## Reviews

Code and architecture reviews are recorded in
[`docs/reviews/LOG.md`](docs/reviews/LOG.md) (§9).

## Updates

### 2026 — created
- Bootstrapped from the `devcontainer-sandbox` copier template.
