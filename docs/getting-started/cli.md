---
comments: true
---

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
  [Authenticating to GitHub Packages](../plugins/claude-code.md#distribution-marketplace-npm).

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

!!! tip
    Findings are recorded into the local store. **Raw secrets never touch disk** — the store keeps only a masked preview of the match and a redacted copy of the file content.


## Review stats

```bash
aka stats              # findings by severity, enforcement actions, latest findings
aka stats --range 7d   # windows ONLY the enforcement section
```

`--range` accepts `7d | 30d | 3m | 6m` and scopes **only** the enforcement aggregates —
findings-by-severity and the latest findings are always all-time. An unrecognized value
silently falls back to `30d` (it doesn't error).

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

