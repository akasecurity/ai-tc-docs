# AKA AI-TC — Docs

Public documentation for [AKA AI-TC](https://github.com/akasecurity/ai-tc) (AI Traffic Control), an open-source control plane that intercepts, inspects, and governs AI prompts and responses.

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). Published to GitHub Pages on every push to `main`.

## Local development

```bash
pip install -r requirements.txt
npm run dev   # or: python3 -m mkdocs serve --dev-addr http://localhost:8000
```

Navigate to `http://localhost:8000`.

## Building

```bash
npm run build   # mkdocs build --strict
```
