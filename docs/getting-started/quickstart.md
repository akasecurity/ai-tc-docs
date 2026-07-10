# Quickstart

This guide takes you from zero to a working AKA installation with the Claude Code plugin active, events being captured, and findings appearing in the local dashboard — no backend, no Docker, no Postgres.

## Prerequisites

- **Node.js 26+** and **pnpm 10+**
- **Claude Code** installed and signed in

## 1. Clone and install

```bash
git clone https://github.com/your-org/ai-control-plane
cd ai-control-plane
pnpm setup   # pnpm install + lefthook hooks
pnpm --filter @alsoknownassecurity/cli build
```

## 2. Set up your local AKA home

```bash
pnpm --filter @alsoknownassecurity/cli exec aka init
```

This creates `~/.aka/settings/settings.json` (your preferences) and
`~/.aka/data/aka.db` (the local SQLite store), seeded with default detection
policies.

## 3. Install the Claude Code plugin

From the repo root, load the plugin for a session:

```bash
pnpm turbo run build --filter=@alsoknownassecurity/plugin-claude-code
claude --plugin-dir ./apps/plugin-claude-code
```

See the [Claude Code plugin guide](../plugin/claude-code.md) for marketplace
distribution and other install paths.

## 4. Test the plugin hook

Run the hook script directly with a sample payload — a fake credential in the
same shape a real secrets rule would match — to verify it works:

```bash
echo '{"prompt":"here is a test credential: see rules/secrets fixtures for the exact shape"}' \
  | node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → {"decision":"block","reason":"AKA blocked this prompt — flagged secrets/aws-access-key ..."}
```

For a concrete example payload, see the fixtures in `rules/secrets/`.

## 5. Open the dashboard

```bash
pnpm --filter @alsoknownassecurity/cli exec aka dashboard
```

Navigate to `http://localhost:4319/security` to see the Events page. The
finding from step 4 should appear once you've triggered it from inside a real
Claude Code session (the direct hook invocation above only tests the script).

## What's next

- [Write your first detection rule](../rules/writing-rules.md)
- [Install the plugin for your team](../plugin/claude-code.md)
- [The `aka` CLI](cli.md) for scanning, stats, and managing detection packs
