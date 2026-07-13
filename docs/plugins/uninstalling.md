---
comments: true
---

# Uninstalling the Plugin

Claude Code and Claude Desktop run the **same plugin package** (`apps/plugin-claude-code`), so uninstalling looks almost identical on both — remove the plugin through the host's own plugin manager, then restart. This page covers both flows and, more importantly, what uninstalling **does and doesn't** clean up.

## Claude Code (CLI)

```bash
claude plugin uninstall aka
```

- If you installed at a non-default scope, pass it explicitly: `-s project` or `-s local` (default is `user`).
- `--prune` also removes any auto-installed dependencies the plugin pulled in that nothing else needs anymore.
- Restart Claude Code afterwards so the running session drops the hooks.

If you don't plan to reinstall, also remove the marketplace registration:

```bash
claude plugin marketplace remove ai-tc
```

This just forgets where to fetch the plugin from — it has no effect on anything already installed or on local AKA data.

## Claude Desktop

Desktop has no terminal, so removal happens in **Settings → Plugins**, mirroring the [install flow](claude-desktop.md#installing):

1. Open **Settings → Plugins**.
2. Find **aka** in the installed plugin list.
3. Click **Uninstall** (or **Remove**).
4. Restart Claude Desktop.

## What uninstalling does — and doesn't — remove

Uninstalling unregisters the plugin from the host (Code or Desktop):

- The four hooks (`SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`) stop firing — prompts and tool calls are no longer scanned, and nothing is warned, redacted, or blocked.
- The `/aka:*` slash commands disappear.

It does **not** touch `~/.aka/` — the settings file, the local SQLite store (`~/.aka/data/aka.db`), the event queue, or the policy cache. That's intentional: Code and Desktop share the same store, and so does the `aka` CLI and dashboard, so uninstalling from one surface doesn't cut the others off from their findings history.

!!! note
    `claude plugin uninstall`'s `--keep-data` flag refers to Claude Code's own per-plugin data directory (`~/.claude/plugins/data/{id}/`). AKA never writes there — all of its state lives under `~/.aka/` — so this flag has no effect on AKA's data either way.

## Fully removing local AKA data (optional)

To also clear everything AKA has stored locally — findings, settings, policy cache, event queue:

```bash
rm -rf ~/.aka
```

!!! warning
    This is shared state: it wipes findings/settings for **both** Claude Code and Claude Desktop, and for the `aka` CLI/dashboard if you use them. Uninstall the plugin from every surface first, or the next session start will just recreate `~/.aka` and start populating it again.

## Reinstalling later

Nothing about your history is lost by uninstalling alone — `~/.aka` persists unless you removed it yourself, so reinstalling and running `/aka:setup` again picks up your existing findings and settings. See [Claude Code → Distribution](claude-code.md#distribution-marketplace-npm) or [Claude Desktop → Installing](claude-desktop.md#installing) for the install steps.
