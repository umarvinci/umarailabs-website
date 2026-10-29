# MD Farooque — Personal Website

Source for my personal portfolio website: who I am, how I can help, case studies and blog posts on RAG, LLM evaluation, machine learning and MLOps.

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). All content is written in Markdown and configured in `mkdocs.yml`.

## Run it locally (macOS)

1. **Install the image libraries** used to generate social preview cards:
   ```bash
   brew install cairo freetype libffi libjpeg libpng zlib pngquant
   ```

2. **Create the virtual environment** (uses Python 3.11 from `.python-version` and the pinned versions in `uv.lock`):
   ```bash
   uv sync
   ```

3. **Start the preview server**:
   ```bash
   export DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/lib
   uv run mkdocs serve
   ```
   Visit `http://localhost:8000`. Add `-a 127.0.0.1:8001` to use a different port.

A Docker setup is also included: run `./start_server.sh`.

## Project structure

```bash
docs/
├── index.md              # Home / About page
├── portfolio/            # Case studies
│   └── projects/
├── blog/                 # Blog
│   ├── .authors.yml
│   └── posts/
├── assets/               # Photo, logo, favicon
└── stylesheets/          # Custom CSS
```

- **New blog post:** add a Markdown file to `docs/blog/posts/` with `date`, `authors: [mdfarooque]` and `categories` in the front matter.
- **New case study:** add a Markdown file to `docs/portfolio/projects/`, link it from `docs/portfolio/index.md` and add it to `nav` in `mkdocs.yml`.

## Deployment

The site is designed to be published with GitHub Pages through GitHub Actions. See [Publishing your site](https://squidfunk.github.io/mkdocs-material/publishing-your-site/).

For a custom domain, put the domain in `docs/CNAME`, set `site_url` in `mkdocs.yml`, then configure DNS following [GitHub's custom domain guide](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site).
