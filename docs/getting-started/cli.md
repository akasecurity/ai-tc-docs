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
- `aka detections [update|unquarantine]` — list installed detection packs, apply updates (manual), clear quarantined rules
- `aka plugins` — list / install agent plugins (Claude Code, Codex, Antigravity, GitHub Copilot)
- `aka dashboard` — launch the local web dashboard and open it in your browser
- `aka extension` — install / check the browser extension's native-messaging host (`install`, `status`, `account`)
- `aka exception` — manage detection exceptions (`approve`, `add`, `list`, `show`, `revoke`, `rotate-key`)
- `aka vault [show|prune]` — reveal a vaulted secret by its pointer (every reveal is audited), or undo vaulting the policy no longer allows
- `aka attach` / `aka detach` / `aka status` — register this machine with your organization's AKA deployment, stop, or show what it's attached to
- `aka enroll` / `aka unenroll` — choose which repositories and accounts a personal-device attachment sends
- `aka sync-history` — whether to send activity recorded before this machine attached
- `aka tui [view]` — interactive terminal dashboard (`health`, `findings`, `recommend`, `audit`)
- `aka check-updates` — show which components (CLI + plugins) have updates available
- `aka update [what]` — update the CLI and/or plugins (`cli`, `<plugin-id>`, `all`)
- `aka completion <zsh|bash>` — emit a shell completion script
- `aka --version` — print the installed CLI version

Global flags:

- `--home <dir>` — use an alternate AKA home (default: `~/.aka`). Accepted by every command
  except `aka dashboard`.
- `--no-update-check` — skip the post-command "updates available" notice
- `-v, --version` / `-h, --help`

On macOS and Linux, any other command runs an external `aka-<command>` from your `PATH` — so
`aka claude` runs `aka-claude`. A built-in command always wins.

---

## Install

The `aka` CLI ships as a **self-contained binary** that embeds its own Node runtime — no
Node.js, no npm, no native dependencies — through Homebrew (`brew install akasecurity/tap/aka`),
Scoop (`scoop install aka` after adding the `akasecurity` bucket), or the bootstrap installer.
It's also on npm as `@akasecurity/cli`. See [Installation](installation.md#install-the-aka-cli)
for every channel. The installer one-liner:

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
  Check with `node --version`. The package is public on the npm registry — no token needed.
  Intel macOS has no binary build, so npm is the install path there.

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

`aka init` is idempotent — run it as often as you like. It also checks for the Claude Code
plugin and offers to install it; pass `--yes` (`-y`) to accept without a prompt.

## Scan for secrets & sensitive data

```bash
aka scan .                          # scan a directory
aka scan path/to/file.env           # scan a single file
aka scan . --format json            # machine-readable result on stdout
aka scan . --fail-on high           # exit 1 if any finding is high or critical
aka scan . --no-forward             # attached machines: record locally only
```

A recursive scan skips build output, vendored code and tool caches (`node_modules`, `dist`,
`build`, `coverage`, `.git`, `.cache`, …) and files over 1 MB. Other dot-directories **are**
entered — except directly under your home directory. Two ignore files control the rest:

- **`.akaignore`** (gitignore syntax, any directory level) is a hard skip: matching files are
  never read and produce no findings. A `!` negation re-includes a directory from the default
  skip list.
- **`.gitignore`**d files are still scanned — generated config is a common hiding place for real
  secrets — but their findings are marked as gitignored.

Some paths are **never** scanned, whatever the ignore files say: `~/.claude/.credentials.json`,
`~/.claude/ide`, `~/.codex/auth.json`, and the AKA home itself. Pointing `aka scan` at one fails
with `refusing to scan <path>: it holds credentials this scanner must never read`.

`--format text|json` picks the output. JSON findings carry the masked match, its span and the
variable it is assigned to — never the raw secret — plus the `inventory`, `egress` and `forward`
results. `--fail-on <critical|high|medium|low>` is for CI. On a machine attached with
`aka attach`, a scan also forwards its Data Shares register and credential sightings to that
deployment; `--no-forward` skips both for one run.

!!! tip
    Findings are recorded into the local store. **Raw secrets never touch disk** — the store keeps only a masked preview of the match and a redacted copy of the file content.


## Review stats

```bash
aka stats              # findings by severity, enforcement actions, latest findings
aka stats --range 30d  # window the enforcement and token-usage sections
```

`aka stats` also reports token usage and estimated cost. `--range` accepts
`7d | 30d | 3m | 6m` (default `7d`) and scopes the enforcement and token-usage sections —
findings-by-severity and the latest findings are always all-time. An unrecognized value
silently falls back to `7d` (it doesn't error).

## Manage detection packs

```bash
aka detections                    # list installed packs + available updates
aka detections update --all       # apply every pending update
aka detections update <pack-id>   # apply one (accepts `secrets` or `aka/secrets`)
aka detections unquarantine       # clear cached ReDoS-quarantine verdicts
```

`aka detections` prints one row per installed pack: installed version, the latest
version this CLI ships, rule count, enabled state, assigned enforcement policy,
and whether an update is available. It also warns about quarantined rules and invalid rules
under enabled packs.

A regex rule that blows the ReDoS timing budget, or has to be stopped mid-scan, is
**quarantined** — cached as unsafe and dropped from later scans. That is a wall-clock judgement,
so a loaded machine can reach it about a rule that is fine; `aka detections unquarantine` clears
the verdicts.

Detection updates are **manual by design**. Upgrading the CLI or plugin only
records what's newly _available_ — the packs you have installed keep running
unchanged (same rules, same versions) until you apply the update yourself with
`aka detections update` or the dashboard's **Update** button. `aka init` and the
plugin's session hooks install packs you don't have yet (new packs arrive
enabled under the policy their manifest names — the `secrets` and `command-risk` packs ship on
_warn_ — or the per-category default otherwise), but they **never modify** an
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
rule that would otherwise catch it. Granting one happens only out-of-band — from the CLI or the
local dashboard's **Exceptions** page — never from inside an agent session, so a compromised
session can't grant itself a bypass.

```bash
aka exception list                  # active grants
aka exception list --all            # + expired, consumed, and revoked (audit evidence)
aka exception show 3f2a91           # full detail (an unambiguous id prefix is enough)
aka exception revoke 3f2a91 --reason "no longer needed"
```

Every grant needs a scope — exactly one of:

- `--once` — single use, with a 30-minute backstop expiry
- `--for <30m|1h|24h>` — time-limited, 24 hours at most
- `--permanent` — until revoked; you confirm by typing the masked value back (or pass `--yes`
  when there's no terminal)

— plus a `--reason` (you're prompted for it on a terminal if you omit it). `--method
<acknowledged|false-positive|test-or-example>` records the reason as a label, and `--yes`
skips confirmations for non-interactive use.

```bash
# Approve something blocked in the last 30 minutes, by the reference in the block message
aka exception approve 3f2a --for 1h --reason "temp deploy creds"

# Pre-authorize a value that wasn't just blocked
aka exception add --rule secrets/aws-access-key --stdin --once --reason "rotating today" < key.txt

# Let the model receive a vaulted value's raw form while the grant is active
aka exception approve '[[aka:secret:...]]' --reveal-to-model --for 1h --reason "agent needs the live key"
```

A vault pointer as the selector requires `--reveal-to-model` (`--reveal` for short): instead of
suppressing a detection, the grant lets the raw value cross tool boundaries, with every crossing
audited.

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

### Undo vaulting the policy no longer allows

A detection assigned Monitor or Warn may only log a match. If you lower a detection's policy,
entries vaulted under the stronger assignment are out of policy:

```bash
aka vault prune            # dry run (default): show what would change
aka vault prune --apply    # restore the transcripts' pointers, then delete those entries
```

It restores the transcript pointers to their raw values **first** and deletes the vault
entries only afterwards — the vault holds the only copy. Entries you asked for yourself (the
Redact button on a surfaced secret) are left alone. `--user-home <dir>` points it at the OS
home whose `~/.claude/projects` holds the transcripts.

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
- **Codex CLI** installs the same way through the `codex` CLI (`codex plugin marketplace add
  akasecurity/ai-tc`, then `codex plugin add aka-codex@ai-tc`).
- **Antigravity** and **GitHub Copilot** install from a local directory rather than a
  marketplace, so `aka plugins install antigravity` / `copilot` prints the steps instead of
  running them.
- A Claude Code plugin your organization's managed settings installed is left alone — the CLI
  refuses to install over it.

`aka plugins list` is read-only — it will **not** create a local store if you haven't
run `aka init` yet.

## Attach to your organization's deployment

By default AKA is standalone. If your organization runs an AKA deployment, `aka attach`
registers this machine with it:

```bash
aka attach --url https://aka.example.com            # prompts for the access key
aka attach --url https://aka.example.com --scoped   # a personal device
aka status                                          # what this machine is attached to
aka detach                                          # stop reporting; record locally only
```

The access key is never a command-line argument — you're prompted for it, or a browser
approval flow is offered, or pipe it with `--key-stdin`. `--label <name>` names the deployment
on screen.

- **`--machine`** — a machine your organization owns: activity from anywhere on it is sent.
- **`--scoped`** — a personal device: only activity from repositories and accounts you enroll,
  and from Claude Code and Codex sessions signed in to a recognised work subscription.

On a scoped attachment, choose what's sent:

```bash
aka enroll                      # the repository this directory is in
aka enroll --repo github.com/acme/payments-api
aka enroll --account <key>      # a Claude account or ChatGPT workspace, as --list-detected shows it
aka enroll --list               # what is enrolled
aka enroll --list-detected      # web-chat accounts the extension saw, and Claude Code / Codex logins
aka unenroll --repo github.com/acme/payments-api
```

Enrolling changes local settings only and contacts nothing. Activity recorded **before** the
attachment is a separate grant:

```bash
aka sync-history          # what's in force
aka sync-history --on     # send it, in the background over later sessions
aka sync-history --off    # don't; stops anything not already sent
```

`aka attach --sync-history` / `--no-sync-history` answers the same question up front. An
attached machine sends more than activity, and what has been sent cannot be recalled —
[SECURITY.md](https://github.com/akasecurity/ai-tc/blob/main/SECURITY.md#data-in-transit) sets
out exactly what each mode sends.

## Expire captured bodies

```bash
aka prune --dry-run       # report what would be freed
aka prune                 # clear prompt/reply/tool/file text past the retention horizon
aka prune --days 14       # override the horizon for this run
```

Every event row and finding is kept; only the raw text is cleared. `aka prune` is a no-op
unless body expiry is switched on in settings (`bodyRetention`, off by default) — it is not a
way to bypass that. It doesn't appear in `aka --help`.

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

