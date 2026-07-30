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

- `aka init` — scaffold your local AKA home (settings + local database)
- `aka scan [path]` — scan a file or directory and record findings (default: `.`)
- `aka stats` — print findings / enforcement / detections from the local store
- `aka detections [update]` — list installed detection packs + apply updates (manual)
- `aka plugins` — list / install agent plugins (Claude Code, …)
- `aka dashboard` — launch the local web dashboard and open it in your browser
- `aka exception` — manage detection exceptions (`approve`, `add`, `list`, `show`, `revoke`, `rotate-key`)
- `aka vault [show]` — reveal a vaulted secret by its pointer (every reveal is audited)
- `aka tui [view]` — interactive terminal dashboard (`health`, `findings`, `recommend`, `audit`)
- `aka check-updates` — show which components (CLI + plugins) have updates available
- `aka update [what]` — update the CLI and/or plugins (`cli`, `<plugin-id>`, `all`)
- `aka completion <zsh|bash>` — emit a shell completion script
- `aka --version` — print the installed CLI version

Global flags, honored for every command:

- `--home <dir>` — use an alternate AKA home (default: `~/.aka`)
- `--no-update-check` — skip the post-command "updates available" notice

---

## Install

The `aka` CLI ships as a **self-contained binary** that embeds its own Node runtime — no
Node.js, no npm, no native dependencies:

```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/akasecurity/ai-tc/bin-latest/tools/installer/install.sh | sh

# Windows (PowerShell)
irm https://raw.githubusercontent.com/akasecurity/ai-tc/bin-latest/tools/installer/install.ps1 | iex
```

The one-liner pins to the latest published binary release tag (`bin-latest`, never `main`)
and verifies the download against that release's `SHA256SUMS` — **fail-closed**, so a
corrupted or tampered download is refused rather than installed. To pin an exact version
instead, set `AKA_INSTALL_REF=bin-v<version>` before running the line.

### Prerequisites

- **Binary install:** none.
- **npm install (`@akasecurity/cli`):** **Node.js 24+** — the CLI uses the built-in
  `node:sqlite` (no native dependency), and the repo tracks the Active LTS line. Install
  from [nodejs.org](https://nodejs.org) or a version manager (`fnm`, `nvm`, `volta`).
  Check with `node --version`.
- **A GitHub token with `read:packages`** — **only while AKA is pre-release**, and only
  for the npm path. The package is published to **private GitHub Packages** during
  stealth, so installing it needs authentication. This requirement goes away when the
  project is public. See
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
aka tui              # pick a view interactively
aka tui health       # jump straight to one
```

`aka tui` opens on the view you name — `health`, `findings`, `recommend`, or `audit`
(the same four surfaces as the `/aka:*` slash commands). It needs a TTY, so it won't
run under a pipe or in CI.

## Manage exceptions

An exception is a deliberate, audited carve-out that lets one specific value through a
rule that would otherwise catch it. Granting one is only available out-of-band from the
CLI — never from inside an agent session, so a compromised session can't grant itself a
bypass.

```bash
aka exception list                  # active grants
aka exception list --all            # + expired, consumed, and revoked (audit evidence)
aka exception show 3f2a91           # full detail (an unambiguous id prefix is enough)
aka exception revoke 3f2a91 --reason "no longer needed"
```

Every grant needs a scope — exactly one of `--once`, `--for <30m|1h|24h>`, or
`--permanent` — plus a `--reason` (you're prompted for it on a terminal if you omit it).

```bash
# Approve something blocked in the last 30 minutes, by the reference in the block message
aka exception approve 3f2a --for 1h --reason "temp deploy creds"

# Pre-authorize a value that wasn't just blocked
aka exception add --rule secrets/aws-access-key --stdin --once --reason "rotating today" < key.txt
```

!!! warning "The value is never an argument"
    `aka exception add` will **not** accept the secret on the command line — pipe it with
    `--stdin` or type it at the hidden prompt. `aka exception approve` does accept a
    pasted value (matched by keyed fingerprint, never stored or echoed), but it lands in
    your shell history — so prefer the reference where that matters.

Nothing is ever deleted: revoked, expired, and consumed grants are kept as audit
evidence. `aka exception rotate-key` mints a new fingerprint key, and **rotation is
invalidation** — every existing grant stops matching, because fingerprints can't be
re-keyed (raw values are never stored). Active permanent grants are listed so you can
re-approve them deliberately.

## Reveal a vaulted secret

When the secret vault is enabled, a detected value can be replaced by an `[[aka:…]]`
**pointer** instead of being destroyed — reversible tokenization rather than one-way
redaction. `aka vault show` resolves a pointer back to its raw value:

```bash
aka vault show '[[aka:...]]'
```

Quote the pointer — the double brackets are shell syntax otherwise. Every successful
reveal writes an audit row to the local store, so de-referencing is never silent.

A pointer that resolves to nothing reports one deliberately indistinguishable failure:
it may be forged or tampered, its entry may have been purged, or the vault key material
may be unavailable. The vault doesn't say which — a token nobody can vouch for resolves
to nothing.

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

