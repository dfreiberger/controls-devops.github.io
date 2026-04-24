# controls-devops.github.io

Source for [controls-devops.github.io](https://controls-devops.github.io/) - documentation for a DevOps workflow built around Beckhoff TwinCAT 3, Azure DevOps, and Ansible.

Built with [MkDocs](https://www.mkdocs.org/) and the [Material](https://squidfunk.github.io/mkdocs-material/) theme.

## Local development

Prerequisites:

- Python 3.10+
- [d2](https://d2lang.com/tour/install) binary on `PATH` (only needed if you render d2 diagrams locally; CI installs it automatically)

Set up a virtual environment and install dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Serve the site locally with live reload:

```bash
mkdocs serve
```

Build a static copy into `site/`:

```bash
mkdocs build --strict
```

## Authoring

- Markdown lives under `docs/`. Nav is defined in `mkdocs.yml`.
- **Mermaid:** use a ` ```mermaid ` fenced code block.
- **d2:** use a ` ```d2 ` fenced code block (rendered to SVG at build time).
- **Math:** inline `\( ... \)`, block `\[ ... \]` (MathJax).
- **Admonitions, code tabs, task lists:** see the [Material reference](https://squidfunk.github.io/mkdocs-material/reference/).

## Deployment

Pushes to `main` trigger `.github/workflows/deploy.yml`, which builds the site and publishes it to GitHub Pages. In the repo settings, **Pages → Source** must be set to **GitHub Actions**.
