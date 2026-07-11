# Quickstart

Get the AKA plugin running in Claude Code, then confirm detection is live — no backend, no Docker, no Postgres.

## Install the plugin

### Point Claude Code at the repo (recommended)

The fastest path — no clone, no build. Run these in the Claude Code **terminal CLI** (`claude`):

```
/plugin marketplace add akasecurity/ai-tc
/plugin install akasecurity@ai-tc
```

Restart Claude Code afterwards to load the plugin.

### claude-tools

[`claude-tools`](https://github.com/akasecurity/claude-tools) can also install the AKA plugin as part of setting up a Claude Code profile — see that repo for the current install command.

### Homebrew — coming soon

### npm — coming soon

## Onboard

Inside a Claude Code session, run the onboarding wizard:

```
/aka:setup
```

This asks three questions (run mode, policy, historical scan consent) and writes your preferences to `~/.aka/settings/settings.json`. See [Configuration](configuration.md) for what each option means.

## Verify it's working

Submit a prompt containing a fake credential — the same shape a real secrets rule would match — and confirm AKA blocks it. See the fixtures in `rules/secrets/` for a concrete example payload.

You should see a block message referencing `secrets/aws-access-key`. Then run `/aka:findings` or `/aka:health` to see it recorded.

## Open the dashboard

```
/aka:dashboard
```

Navigate to `http://localhost:4319/security` to see the Events page.

## What's next

- [Write your first detection rule](../rules/writing-rules.md)
- [The `aka` CLI](cli.md) for scanning, stats, and managing detection packs
- [Claude Code plugin guide](../plugin/claude-code.md) for hook internals and configuration details

---

## Build from source (contributors)

Building AKA from source instead of installing the published plugin — useful if you're developing AKA itself.

**Prerequisites:** Node.js 26+ and pnpm 10+, Claude Code installed and signed in.

### 1. Clone and install

```bash
git clone https://github.com/akasecurity/ai-tc
cd ai-tc
pnpm setup   # pnpm install + lefthook hooks
pnpm --filter @akasecurity/cli build
```

### 2. Set up your local AKA home

```bash
pnpm --filter @akasecurity/cli exec aka init
```

This creates `~/.aka/settings/settings.json` (your preferences) and
`~/.aka/data/aka.db` (the local SQLite store), seeded with default detection
policies.

### 3. Load the plugin for a session

```bash
pnpm turbo run build --filter=@akasecurity/plugin-claude-code
claude --plugin-dir ./apps/plugin-claude-code
```

### 4. Test the plugin hook

Run the hook script directly with a sample payload — a fake credential in the
same shape a real secrets rule would match — to verify it works:

```bash
echo '{"prompt":"here is a test credential: see rules/secrets fixtures for the exact shape"}' \
  | node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → {"decision":"block","reason":"AKA blocked this prompt — flagged secrets/aws-access-key ..."}
```

For a concrete example payload, see the fixtures in `rules/secrets/`.

### 5. Open the dashboard

```bash
pnpm --filter @akasecurity/cli exec aka dashboard
```

Navigate to `http://localhost:4319/security` to see the Events page. The
finding from step 4 should appear once you've triggered it from inside a real
Claude Code session (the direct hook invocation above only tests the script).
