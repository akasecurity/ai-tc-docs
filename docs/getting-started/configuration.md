# Configuration

The OSS surface — the Claude Code plugin, the `aka` CLI, and the web-ui — has
**no environment-variable configuration**. It reads and writes local files
under `~/.aka` instead:

- `~/.aka/settings/settings.json` — onboarding preferences (`runMode`,
  `policy`, `historicalAccess`), written by `/aka:setup` or `aka init`. See
  [Claude Code plugin → Configuration](../plugin/claude-code.md#configuration).
- `~/.aka/data/aka.db` — the local SQLite store the plugin and CLI write to
  (events, findings, policies).

This is deliberate: hooks are bare processes Claude Code spawns, so
environment variables set in your shell can't reach them — a file is the only
reliable channel.

