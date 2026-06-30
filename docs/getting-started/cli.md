# The `aka` CLI

`aka` is the local-first command-line tool for AKA. It runs **entirely on your
machine** — detection, the local store, and the dashboard all work with **no backend
and no Docker**. It reads and writes the same local SQLite store (`~/.aka/data/aka.db`)
that the Claude Code plugin uses, so the CLI and the plugin share one view of your
findings.

What you get:

- `aka init` — set up your local AKA home
- `aka scan` — scan files/directories for secrets & sensitive data
- `aka stats` — print findings/enforcement aggregates
- `aka dashboard` — open the local web dashboard
- `aka tui` — an interactive terminal dashboard
- `aka plugins` — install/manage agent plugins (Claude Code, …)
- `aka attach` — (scaffold) point a standalone setup at an enterprise platform

---

## Prerequisites

- **Node.js 26** — the CLI uses the built-in `node:sqlite` (no native dependency), and
  the repo pins `engines: 26.x`. Install from [nodejs.org](https://nodejs.org) or a
  version manager (`fnm`, `nvm`, `volta`). Check with `node --version`.
- **A GitHub token with `read:packages`** — **only while AKA is pre-release.** The CLI
  is published to **private GitHub Packages** during stealth, so installing it needs
  authentication. This requirement goes away when the project is public. See
  [Authenticating to GitHub Packages](#authenticating-to-github-packages-pre-release).

---

## Install (recommended: the bootstrap installer)

> **While AKA is pre-release (private repo):** the `curl | sh` / `irm | iex` one-liner
> fetches the installer from `raw.githubusercontent.com`, which returns **404 for a
> private repo unless the request is authenticated**. During stealth, use the
> [manual install](#manual-install-if-youd-rather-not-use-the-one-liner) below
> (`npm i -g` with a `read:packages` token) — it works today because the package is
> published to GitHub Packages. The one-liner becomes the primary path once the repo
> is public (or authenticate the fetch with a `repo`-scoped token:
> `curl -fsSL -H "Authorization: token $GITHUB_TOKEN" …`).

The one-liner **requires Node 26** — it checks for it and tells you to install it if
it's missing; it does **not** install Node for you — then configures the package
registry and installs the global `aka` CLI. Provide your GitHub token via the
`GITHUB_TOKEN` environment variable (this keeps it out of your shell history; the
installer writes it to `~/.npmrc` with mode `0600`):

=== "macOS / Linux"

    ```bash
    export GITHUB_TOKEN=ghp_your_read_packages_token
    curl -fsSL https://raw.githubusercontent.com/alsoknownassecurity/ai-control-plane/cli-v0.0.1/tools/installer/install.sh | sh
    ```

=== "Windows (PowerShell)"

    ```powershell
    $env:GITHUB_TOKEN = "ghp_your_read_packages_token"
    irm https://raw.githubusercontent.com/alsoknownassecurity/ai-control-plane/cli-v0.0.1/tools/installer/install.ps1 | iex
    ```

> **Note — why the URL is pinned to `cli-v0.0.1`:** the one-liner fetches the
> installer from an **immutable release tag**, never `main`, and verifies the
> downloaded installer script (`install.mjs`) against a SHA-256 baked into the shell
> script (`install.sh`) before running it. Bump the tag to the latest release as new
> versions ship. (Before the first `cli-v*` tag exists, the URL 404s by design.)

Then verify:

```bash
aka --help
```

### Manual install (if you'd rather not use the one-liner)

1. Configure `~/.npmrc` (see [below](#authenticating-to-github-packages-pre-release)).
2. Install the global package:

   ```bash
   npm install -g @alsoknownassecurity/cli
   ```

---

## Authenticating to GitHub Packages (pre-release)

While the packages are private, `npm`/`aka` need to know (a) that the
`@alsoknownassecurity` scope lives on GitHub Packages and (b) a token to read it. The
bootstrap installer does this for you; to set it up by hand, add these two lines to
your **`~/.npmrc`**:

```ini
@alsoknownassecurity:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=ghp_your_read_packages_token
```

**Where to get the token:**

1. GitHub → **Settings → Developer settings → Personal access tokens → Tokens
   (classic)** → **Generate new token (classic)**.
2. Give it the **`read:packages`** scope (nothing else is needed to install).
3. Copy the token (starts with `ghp_`) into the `_authToken` line above, or export it
   as `GITHUB_TOKEN` before running the installer.

> **⚠️ Keep the token private:** pass it via the environment (`GITHUB_TOKEN`) rather
> than typing it on the command line, so it doesn't land in your shell history.
> `~/.npmrc` should be readable only by you (`chmod 600 ~/.npmrc`); the installer
> sets that automatically.

---

## Set up your machine

```bash
aka init
```

This scaffolds your local AKA home (owner-only `~/.aka`):

- `~/.aka/settings/settings.json` — your preferences (run mode, redaction policy).
  Re-running `aka init` **never overwrites** an existing settings file.
- `~/.aka/data/aka.db` — the local SQLite store (created, migrated, and seeded with the
  default per-category detection policies).

`aka init` is idempotent — run it as often as you like.

---

## Scan for secrets & sensitive data

```bash
# Scan a directory (skips node_modules, ANY dot-directory, build output, files > 1 MB)
aka scan .

# Scan a single file (no skip rules — works even for dot-dirs like ~/.aws, ~/.ssh)
aka scan path/to/file.env
aka scan ~/.aws/credentials
```

A recursive `aka scan .` skips **every hidden (dot-)directory** — `.git`, but also
`.github`, `.aws`, `.ssh`, `.config`, … — so it won't descend into them. To sweep a
dot-directory, point `aka scan` **directly** at the file or folder (the single-path
form has no skip rules).

Findings are recorded into the local store. **Raw secrets never touch disk** — the
store keeps only a masked preview of the match and a redacted copy of the file
content.

```bash
aka stats              # findings by severity, enforcement actions, latest findings
aka stats --range 7d   # windows ONLY the enforcement section
```

`--range` accepts `7d | 30d | 3m | 6m` and scopes **only** the enforcement aggregates —
findings-by-severity and the latest findings are always all-time. An unrecognized value
silently falls back to `30d` (it doesn't error).

---

## Open the dashboard from the CLI

```bash
aka dashboard
```

This launches the local web dashboard (the OSS Next.js app) against your `~/.aka`
store and opens your browser at `http://localhost:4319/security`. It reads the local
store directly — **no backend, no auth, nothing leaves your machine.**

```bash
aka dashboard --port 8080   # use a different port
aka dashboard --no-open     # start the server without opening a browser
```

Prefer the terminal? Use the interactive Ink dashboard instead:

```bash
aka tui
```

---

## Install agent plugins

The CLI is an optional **hub** for installing agent plugins — but each plugin also
installs on its own, so the CLI is never required.

```bash
aka plugins list                 # show available agents + which are active locally
aka plugins install claude-code  # install / set up an agent plugin
```

- **Claude Code** is distributed through the **AKA marketplace** (which you add inside
  Claude Code), not npm. `aka plugins install claude-code` doesn't install anything
  itself yet — it prints how to add the AKA marketplace in Claude Code; then run
  `aka init`.
- Other agents (Cursor, GitHub Copilot, …) appear as **coming soon** until they ship.

`aka plugins list` is read-only — it will **not** create a local store if you haven't
run `aka init` yet.

---

## Attach to an enterprise platform (preview)

```bash
aka attach https://your-org.aka.example --org your-org-id
```

`aka attach` is currently a **scaffold**: it flips your local config to attached mode
and records the binding. Your local store stays **tenant-free** — tenant identity is
assigned server-side at ingest, never written locally. The full auth handshake + data
sync land in a later release.

---

## Where things live

| Path                            | What                                                |
| ------------------------------- | --------------------------------------------------- |
| `~/.aka/settings/settings.json` | Your preferences (run mode, redaction policy)       |
| `~/.aka/data/aka.db`            | The local SQLite store (events, findings, policies) |
| `~/.npmrc`                      | GitHub Packages scope + auth (pre-release only)     |

Everything is local. To start over, remove `~/.aka` and run `aka init` again.
