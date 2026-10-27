---
comments: true
---

# Uninstalling the Plugin

Every AKA plugin is removed through its own host's plugin manager, followed by a restart. Claude
Code and Claude Desktop run the **same plugin package** (`plugins/claude-code`), so those two
flows are near-identical; Codex, Antigravity, GitHub Copilot, and the browser extension each have their own. This
page covers all of them and, more importantly, what uninstalling **does and doesn't** clean up.

## Claude Code (CLI)

```bash
claude plugin uninstall aka
```

- If you installed at a non-default scope, pass it explicitly: `-s project` or `-s local` (default is `user`).
- `--prune` also removes any auto-installed dependencies the plugin pulled in that nothing else needs anymore.
- Restart Claude Code afterwards so the running session drops the hooks.

If you don't plan to reinstall, also remove the marketplace registration:

```bash
claude plugin marketplace remove akasecurity
```

This just forgets where to fetch the plugin from — it has no effect on anything already installed or on local AKA data.

## Codex CLI

```bash
codex plugin uninstall aka-codex@ai-tc
codex plugin marketplace remove ai-tc   # optional, if you won't reinstall
```

Restart Codex afterwards so the running session drops the hooks. The `aka-*` skills disappear with
the plugin; `~/.aka` is untouched.

## Antigravity

Antigravity installed the plugin from a local directory, so remove it with `agy`'s own plugin
manager and restart the CLI. The extracted `./package` directory you installed from is just a
staging copy — deleting it after removal is safe (and deleting it *before* removal doesn't
uninstall anything).

## GitHub Copilot

```bash
copilot plugin uninstall aka-copilot
copilot plugin list   # aka-copilot should no longer appear
```

For VS Code agent mode, also remove the folder's entry from `chat.pluginLocations` in your user
settings. The `aka-copilot` folder you installed from is a staging copy — safe to delete once the
plugin is removed from every surface that loads it.

## Browser extension

Two halves to remove:

1. Remove **AI Traffic Control** from `chrome://extensions`.
2. Delete the native-messaging manifest and launcher from the directory `aka extension status`
   reports. On Windows also remove the registry key
   `HKCU\Software\Google\Chrome\NativeMessagingHosts\com.akasecurity.aka`.

Leaving the host installed with no extension is harmless — Chrome only spawns it for the pinned
extension id — but it will keep answering if you reload the extension later.

## Claude Desktop

Desktop has no terminal, so removal happens in **Settings → Plugins**, mirroring the [install flow](claude-desktop.md#installing):

1. Open **Settings → Plugins**.
2. Find **aka** in the installed plugin list.
3. Click **Uninstall** (or **Remove**).
4. Restart Claude Desktop.

## What uninstalling does — and doesn't — remove

Uninstalling unregisters the plugin from that one host:

- Its hooks stop firing — prompts and tool calls are no longer scanned there, and nothing is warned, redacted, or blocked.
- Its read surface disappears: the `/aka:*` slash commands on Claude Code/Desktop, the `aka-*` skills on Codex and Antigravity, the `aka-setup` skill on Copilot.

It does **not** touch `~/.aka/` — the settings file, the local SQLite store (`~/.aka/data/aka.db`), the event queue, or the policy cache. That's intentional: every plugin shares that store, and so do the `aka` CLI and the dashboard, so uninstalling from one surface doesn't cut the others off from their findings history. It also doesn't disable the *other* hosts: findings from a still-installed Codex or Antigravity plugin keep arriving.

!!! note
    `claude plugin uninstall`'s `--keep-data` flag refers to Claude Code's own per-plugin data directory (`~/.claude/plugins/data/{id}/`). AKA never writes there — all of its state lives under `~/.aka/` — so this flag has no effect on AKA's data either way.

## Fully removing local AKA data (optional)

To also clear everything AKA has stored locally — findings, settings, policy cache, event queue:

```bash
rm -rf ~/.aka
```

!!! warning
    This is shared state: it wipes findings/settings for **every** surface — Claude Code, Claude Desktop, Codex, Antigravity, GitHub Copilot, the browser extension, and the `aka` CLI/dashboard. Uninstall the plugin from every surface first, or the next session start will just recreate `~/.aka` and start populating it again.

## Reinstalling later

Nothing about your history is lost by uninstalling alone — `~/.aka` persists unless you removed it yourself, so reinstalling and running `/aka:setup` again picks up your existing findings and settings. See [Claude Code → Distribution](claude-code.md#distribution-marketplace-npm) or [Claude Desktop → Installing](claude-desktop.md#installing) for the install steps.
