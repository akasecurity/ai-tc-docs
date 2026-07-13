
## Build from source (contributors)

Building AKA from source instead of installing the published plugin — useful if you're developing AKA itself. This workflow uses the Claude Code **terminal CLI**; there's no equivalent for Desktop, since it depends on shell access.

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
