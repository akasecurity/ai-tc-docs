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
- `aka detections` — list installed detection packs + apply updates (manual)
- `aka dashboard` — open the local web dashboard
- `aka tui` — an interactive terminal dashboard
- `aka plugins` — install/manage agent plugins (Claude Code, …)
- `aka check-updates` — see whether the CLI or your plugins have updates
- `aka update` — update the CLI and/or plugins to the latest version
- `aka attach` — (scaffold) point a standalone setup at an enterprise platform
- `aka --version` — print the installed CLI version

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

## Manage detection packs

```bash
aka detections                    # list installed packs + available updates
aka detections update --all       # apply every pending update
aka detections update <pack-id>   # apply one (accepts `secrets` or `aka/secrets`)
```

`aka detections` prints one row per installed pack: installed version, the latest
version this CLI ships, rule count, enabled state, assigned enforcement policy,
and whether an update is available.

Detection updates are **manual by design**. Upgrading the CLI or plugin only
records what's newly _available_ — the packs you have installed keep running
unchanged (same rules, same versions) until you apply the update yourself with
`aka detections update` or the dashboard's **Update** button. `aka init` and the
plugin's session hooks install packs you don't have yet (new packs arrive
enabled under the log-only _monitor_ policy), but they **never modify** an
installed pack; your enabled/disabled choices and policy assignments always
survive an update.

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
aka plugins list                 # show available agents, installed version, active state
aka plugins install claude-code  # install / set up an agent plugin
```

- **Claude Code** is distributed through the **AKA marketplace**, and
  `aka plugins install claude-code` installs it end-to-end: it adds the marketplace
  and installs the plugin by delegating to the `claude` CLI's plugin manager, then
  reminds you to restart Claude Code and run `aka init`. If the `claude` CLI isn't on
  your `PATH`, it falls back to printing the in-app `/plugin` commands to run instead.
- Other agents (Cursor, GitHub Copilot, …) appear as **coming soon** until they ship.

`aka plugins list` is read-only — it will **not** create a local store if you haven't
run `aka init` yet.

---

## Keeping up to date

The CLI and your installed plugins are versioned independently. Two commands keep
them current:

```bash
aka check-updates   # read-only: show installed vs latest for the CLI + each plugin
aka update          # update everything that's behind (asks before applying)
aka update cli      # update just the CLI
aka update claude-code   # update just one plugin
```

- `aka check-updates` changes nothing — it just reports what's available. Latest
  versions are resolved with `npm view` through your existing `~/.npmrc` auth (the
  same one that installed the packages); if the registry is unreachable, "Latest"
  shows as **unknown** and nothing is flagged.
- `aka update` shows what would change and **prompts for confirmation** before
  applying. Pass `--yes` (or `-y`) to skip the prompt (required when there's no
  interactive terminal, e.g. in a script). The CLI updates itself with
  `npm install -g @alsoknownassecurity/cli@latest`; plugins update through the `claude` plugin
  manager (**restart Claude Code afterwards** to load the new version).
- After other commands, the CLI prints a one-line **notice** when an update — or a
  newly available plugin — is waiting, with the exact command to run. It's computed
  from a once-a-day cached check (refreshed in the background, never blocking), is
  suppressed when output isn't a terminal, and can be turned off per run with
  `--no-update-check`.

---

## Attach to an enterprise platform (preview)

`aka attach` is currently a **scaffold** for pointing a standalone setup at an
enterprise platform. See the enterprise docs for the full auth handshake and
data-sync model once it lands.

---

## Shell tab-completion

Turn on `<TAB>` completion for `aka` — type `aka exc<TAB>` and your shell fills in
`aka exception`.

```bash
# zsh (macOS default)
echo 'source <(aka completion zsh)' >> ~/.zshrc

# bash
echo 'source <(aka completion bash)' >> ~/.bashrc
```

Open a new terminal (or `source` the same line now) and `aka <TAB>` completes
commands, subcommand verbs (`aka exception <TAB>` → `approve add list …`),
`aka scan` file paths, and global flags. `aka completion <zsh|bash>` just prints the
script — you load it into your shell, you don't read it.

---

## Where things live

| Path                            | What                                                |
| ------------------------------- | --------------------------------------------------- |
| `~/.aka/settings/settings.json` | Your preferences (run mode, redaction policy)       |
| `~/.aka/data/aka.db`            | The local SQLite store (events, findings, policies) |
| `~/.npmrc`                      | GitHub Packages scope + auth (pre-release only)     |

Everything is local. To start over, remove `~/.aka` and run `aka init` again.
