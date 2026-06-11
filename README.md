# AIB-Docs

## Prerequisites

- Python 3.10+ (recommended)
- `pip` (or `uv`)

## Run locally (pip)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install zensical
zensical serve
```

Open http://localhost:8000 in your browser.

## Run locally (uv)

```bash
uv init
uv add --dev zensical
uv run zensical serve
```

## Build static site

```bash
zensical build --clean
```

The generated static site will be in the `site/` directory.

## Deploy to GitHub Pages

1. Push this repository to GitHub.
2. In GitHub, go to **Settings > Pages** and set **Source** to **GitHub Actions**.
3. Update `site_url` in `zensical.toml`:

```toml
site_url = "https://<username>.github.io/<repository>/"
```

4. Push to `main` (or `master`).

The workflow in `.github/workflows/docs.yml` builds and deploys automatically.

## Customize

- Edit homepage content in `docs/index.md`
- Add pages in `docs/`
- Configure project metadata and nav in `zensical.toml`
