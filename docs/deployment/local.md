# Local / Single-Node

## OSS single-node (no server, no Docker, no Postgres)

The open-source surface needs **no backend at all**. The Claude Code plugin (and
the `aka` CLI) write a local SQLite store at `~/.aka/data/aka.db` via
`@alsoknownassecurity/persistence` (Node's built-in `node:sqlite` — no native dependency), and
the OSS web dashboard reads that same store directly in Server Components. Nothing
leaves the machine; there is no account, no network, no database server.

```bash
# The plugin self-installs and writes ~/.aka/data/aka.db during Claude Code sessions.
# Browse your local findings/health with the OSS dashboard:
aka dashboard        # launches the Next.js web-ui over ~/.aka/data
# or a terminal view:
aka tui
```

See the [CLI guide](../getting-started/cli.md) and the
[Claude Code plugin](../plugin/claude-code.md) page. This is the recommended
single-user local experience.

For running the enterprise control plane locally (against Postgres), see the
enterprise docs.
