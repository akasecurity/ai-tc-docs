---
comments: true
---

# Installation

## Install the plugin

=== "Claude Code"

    The fastest path — no clone, no build. Run these in the Claude Code **terminal CLI** (`claude`):

    ```
    /plugin marketplace add akasecurity/marketplace
    /plugin install ai-tc@akasecurity
    ```

    Restart Claude Code afterwards to load the plugin.

    [`claude-tools`](https://github.com/akasecurity/claude-tools) can also install the AKA plugin as part of setting up a Claude Code profile — see that repo for the current install command.

=== "Claude Desktop"

    Claude Desktop has no terminal, so install from the GUI instead:

    1. Open **Settings → Plugins**.
    2. **Add marketplace** and enter `akasecurity/marketplace`.
    3. Find **ai-tc** in the list and click **Install**.
    4. Restart Claude Desktop.

    See the [Claude Desktop plugin guide](../plugins/claude-desktop.md) for the full walkthrough.

=== "Codex CLI"

    Codex resolves the plugin from the `akasecurity/ai-tc` repo's own marketplace file:

    ```bash
    codex plugin marketplace add akasecurity/ai-tc
    codex plugin add aka-codex@ai-tc
    ```

    Codex has no `plugin install` verb — `add` both installs and updates.

    Restart Codex afterwards. See the [Codex plugin guide](../plugins/codex.md) for the hook
    contract and the skill-based read surface.

=== "Antigravity"

    Antigravity has no plugin marketplace — `agy plugin install` takes a local directory:

    ```bash
    npm pack @akasecurity/ai-tc-antigravity
    tar -xzf akasecurity-ai-tc-antigravity-*.tgz
    agy plugin install ./package
    aka init
    ```

    Hooks fire in the `agy` **CLI** only, and this host cannot block or redact prompts — read
    [Known limitations](../plugins/antigravity.md#known-limitations) before relying on it for
    enforcement.

=== "Browser (ChatGPT / Claude.ai)"

    Web chat is covered by a Chrome extension plus a local native-messaging host:

    ```bash
    aka extension install
    ```

    Then load the built extension via **Load unpacked** at `chrome://extensions` (the command
    prints the path). See the [browser extension guide](../plugins/browser-extension.md).

!!! tip "Already have the `aka` CLI?"
    `aka plugins list` shows every supported agent and whether it's installed, and
    `aka plugins install <agent>` (`claude-code`, `codex`, `antigravity`) runs the host's own
    plugin manager for you. Every plugin writes the same local store, so mixing agents just means
    more of your activity lands in one findings history.

## Install the `aka` CLI

The plugins work without it, but the `aka` CLI adds [scanning, stats, exceptions, updates, and
the dashboard](cli.md). Pick one channel:

=== "npm"

    ```bash
    npm install -g @akasecurity/cli
    ```

    Requires **Node.js 24+** (the CLI uses the built-in `node:sqlite`). This is also the only
    option on Intel macOS, which has no standalone binary build.

=== "Homebrew (macOS / Linux)"

    ```bash
    brew install akasecurity/tap/aka
    ```

    A self-contained binary — no Node.js needed.

=== "Scoop (Windows)"

    ```powershell
    scoop bucket add akasecurity https://github.com/akasecurity/scoop-bucket
    scoop install aka
    ```

=== "Installer script"

    ```bash
    # macOS / Linux
    curl -fsSL https://raw.githubusercontent.com/akasecurity/ai-tc/bin-latest/tools/installer/install.sh | sh
    ```

    ```powershell
    # Windows (PowerShell)
    irm https://raw.githubusercontent.com/akasecurity/ai-tc/bin-latest/tools/installer/install.ps1 | iex
    ```

    Downloads the same standalone binary archive Homebrew and Scoop use. Direct archive
    downloads, `SHA256SUMS`, and build-provenance verification are described in the
    [installer README](https://github.com/akasecurity/ai-tc/blob/main/tools/installer/README.md).

Then run `aka init` to create the local store. Upgrade the same way you installed — see
[Updates](updates.md).

## Onboard

Inside a session, run the onboarding wizard:

```
/aka:setup
```

On Codex and Antigravity there is no `/aka:*` command namespace — the same wizard ships as the
**`aka-setup` skill**; ask for it by name in the session.

The wizard calibrates AKA from your *actual* activity rather than a demo profile: it
offers one retroactive scan of the last 30 days of Claude history, reports the real
numbers it found, recommends a per-category detection posture, and **writes nothing
until you confirm**. Your preferences land in `~/.aka/settings/settings.json` (created
`0600`, owner-only).

It asks for **two separate consents**, and you can decline either one and still finish
setup:

1. **Historical access** — may AKA *read* surfaces that existed before it was installed
   (prior conversation transcripts, scratch/temp files, agent memory)? Declining limits
   AKA to the live session, working tree, git history, and pointed scans.
2. **Model-judge consent** — may AKA *send* what that scan found to the model API to be
   rated? See [Why the setup scan sends findings to the
   model](#why-the-setup-scan-sends-findings-to-the-model) below. Declining skips the
   triage and starts you on the conservative severity floor instead.

Both are revocable from the dashboard under **Settings → Historical access** and
**Settings → Model-judge consent**.

### Why the setup scan sends findings to the model

AKA is local-first: detection runs in-process, findings persist to SQLite on your disk,
and there is no AKA backend or account. The `/aka:setup` triage is the **one deliberate
exception**, and it is opt-in.

**Why it exists.** A raw pattern match can't tell a live production key from a test
fixture, a placeholder in a code comment, or an example in documentation. A scanner that
can't make that call has two bad options: scream about everything, until you learn to
ignore it, or stay quiet and miss the real leak. So AKA asks a model to rate each finding
— real leak or routine noise, and how severe — and uses those ratings to calibrate your
posture and to propose which false positives to suppress. You review the masked evidence
and approve that list before anything is written. Rating a redacted value would defeat
the purpose: judging whether an `AKIA…` string is real requires seeing it.

**What crosses the wire.** Per finding, exactly two things:

- its **raw, unmasked value**, and
- roughly **120 characters of surrounding transcript text** on either side, with any
  *other* secret AKA's rules detect in that window masked first.

The **file path is never sent.** Neither is the value's fingerprint or key version.
Ordinary prose in that context window travels as-is.

**Where it goes.** To the same model provider your Claude session already uses, through
`claude -p` subprocesses — one per batch, so a large history is several calls. Not to
AKA: we run no service that receives this, and we store none of it off your machine.

**What it is not.** Those subprocesses ask the `claude` CLI to write no transcript, so
the raw values never enter your conversation or your own scannable history. That is
*transcript* isolation, not *network* isolation — a copy of each value leaves the
machine. We won't describe it as staying "inside an isolated subprocess," because it
doesn't.

**Revocation has a limit.** Turning the grant off under **Settings → Model-judge
consent** stops future scans. It **cannot recall what was already sent.** If a live key
was in that payload, treat it as exposed and rotate it — which is exactly what the
wizard's remediation step offers to help you do.

**Consent is scoped to a payload shape.** Your grant records the payload version it was
given against. If a release ever widens what crosses, the old grant stops counting and
AKA asks again rather than holding you to terms you never saw.

!!! info "Prefer to keep it fully local?"
    Choose **No, keep it local** at the model-judge prompt. AKA starts from the
    conservative severity floor — high-impact categories at `warn`, observe-only ones at
    `monitor` — and tunes from there as it watches live activity. Nothing leaves your
    machine. Re-run `/aka:setup` any time to change your mind.

## Verify it's working

Submit a prompt containing a fake credential — the same shape a real secrets rule would match — and confirm AKA blocks it. See the fixtures in `rules/secrets/` for a concrete example payload.

You should see a block message referencing `secrets/aws-access-key`. Then run `/aka:findings` or `/aka:health` to see it recorded.

## Open the dashboard

```
/aka:dashboard
```

Navigate to `http://localhost:4319/security` to see the Security page.

## What's next

- Review the findings inside your harness with [`/aka:findings`](usage.md#akafindings)
- Use [the `aka` CLI](cli.md) for scanning, stats, and managing detection packs
- Per-host hook internals and configuration details: [Claude Code](../plugins/claude-code.md),
  [Claude Desktop](../plugins/claude-desktop.md), [Codex CLI](../plugins/codex.md),
  [Antigravity](../plugins/antigravity.md), [browser extension](../plugins/browser-extension.md)
