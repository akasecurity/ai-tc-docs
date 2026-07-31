# AGENTS.md — ai-tc-docs

Per-repo conventions for any coding/ops agent. Builds on `~/aka/AGENTS.md` (company layer) and the
global layer — never repeats them.

## What this repo is

Two jobs in one public repo:

1. **The documentation site for AI Traffic Control** (`akasecurity/ai-tc`), built with MkDocs
   Material and published to GitHub Pages at
   <https://akasecurity.github.io/ai-tc-docs/> on every push to `main`.
2. **The GitHub Discussions host** backing giscus page comments (`overrides/partials/comments.html`).
   That is why Discussions are enabled here and not on the product repo.

Public, and it deploys on merge — there is no staging step between a push and the live site.

## Local development

```bash
pip install -r requirements.txt
npm run dev        # or: python3 -m mkdocs serve --dev-addr http://localhost:8000
```

`site_url` reads `DOCS_SITE_URL` and falls back to the Pages URL, so a local build needs no config.

## The product docs, not the product

The source of truth for behaviour is `akasecurity/ai-tc`. When documenting a flag, a rule pack, or
an egress path, **read it there and match it** — the README's egress footnote in particular is
precise and load-bearing, so paraphrasing it loosely is a correctness bug, not a style choice.

`overrides/home.html` is hand-written HTML sitting outside the Markdown docs, so it is easy to
forget in a sweep. It carries feature cards that link to **sibling repos**, and it went stale
through the `flightcrew` → `preflight` rename: the card advertised "Flightcrew Skills" pointing at
a repo that had been renamed, live, for eleven days. GitHub's redirect kept the link working, which
is exactly why nobody noticed.

Before editing any cross-repo link or install command here, check the real name:

```bash
gh api repos/akasecurity/<repo> --jq .full_name
rg -n '"name"' ../marketplace/.claude-plugin/marketplace.json
```

## Workflow

Contributions land through PRs (the repo has taken numbered PRs throughout). Keep the mkdocs nav and
the file tree in sync; a page that exists but isn't in `nav` is unreachable from the site chrome.
