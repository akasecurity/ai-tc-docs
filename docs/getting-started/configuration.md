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

## Modes

`runMode` in `settings.json` picks the plugin's data path:

- **`standalone`** (default) — reads/writes the local SQLite store directly
  via `@alsoknownassecurity/persistence`. No server, no network, no account.
- **`attached`** — points the plugin at an enterprise backend over its HTTP
  API instead. Setting up and configuring the enterprise backend itself
  (Postgres, Better Auth, multi-tenancy) is covered in the enterprise docs,
  not here.

## Generating a local token

If you're running the optional local backend (`aka` CLI's bundled server) in
`local`/`test` mode, it authenticates with a shared bearer token:

```bash
openssl rand -hex 32
```

Use the output as `AKA_LOCAL_TOKEN`, set identically in both the backend and
the Claude Code plugin environment.
