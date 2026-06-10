# Claude Code Plugin

The Claude Code plugin (`apps/plugin-claude-code`) hooks into Claude Code sessions using the official hook system. It captures every prompt, tool invocation, and response, scans for sensitive data, and forwards events to the AKA backend.

## How it works

Claude Code exposes three hook points that AKA uses:

| Hook               | File                    | Fires when                                                   |
| ------------------ | ----------------------- | ------------------------------------------------------------ |
| `UserPromptSubmit` | `user-prompt-submit.js` | User presses Enter on a prompt                               |
| `PreToolUse`       | `pre-tool-use.js`       | Before Claude executes a tool (read file, run command, etc.) |
| `PostToolUse`      | `post-tool-use.js`      | After a tool returns its result                              |

Each hook is a standalone Node.js script that:

1. Reads a JSON payload from `stdin`
2. Runs the detection engine + policy resolution
3. Queues the event to the backend (fire-and-forget, non-blocking)
4. Writes `{action, prompt?, reason?}` to `stdout`
5. Exits

The entire execution is wrapped in `try/catch`. If anything goes wrong — backend is down, network error, rule crash — the hook **always** returns `{action: "allow"}`. The plugin must never break a developer's session.

## Hook output format

Claude Code reads the hook's stdout to decide what to do next.

| Field    | Type                   | Description                                      |
| -------- | ---------------------- | ------------------------------------------------ |
| `action` | `"allow"` \| `"block"` | Whether to let the prompt/tool proceed           |
| `prompt` | `string?`              | The (possibly redacted) prompt to send to Claude |
| `reason` | `string?`              | Shown to the user if `action` is `"block"`       |

Example — prompt passes through unchanged:

```json
{ "action": "allow", "prompt": "Help me refactor this function" }
```

Example — secret redacted before Claude sees it:

```json
{ "action": "allow", "prompt": "My AWS key is [REDACTED:SECRET], can you check this config?" }
```

Example — blocked by policy:

```json
{
  "action": "block",
  "reason": "Prompt contains a secret (AWS access key). Remove credentials before sharing with AI tools."
}
```

## Installation

### 1. Build the hook scripts

```bash
pnpm --filter @aka/plugin-claude-code build
# Outputs: apps/plugin-claude-code/scripts/*.js
```

### 2. Configure hooks.json

The hook configuration is at `apps/plugin-claude-code/hooks/hooks.json`. Claude Code reads this file to discover hook scripts.

Register it with Claude Code:

```bash
# Add to your Claude Code user settings or project settings
# (exact method depends on your Claude Code version)
claude config add hooks --file apps/plugin-claude-code/hooks/hooks.json
```

Or copy the plugin directory into Claude Code's plugin path:

```bash
cp -r apps/plugin-claude-code/.claude-plugin ~/.claude/plugins/aka/
```

### 3. Set environment variables

The plugin reads two variables. Set them before starting Claude Code:

```bash
export AKA_BACKEND_URL=http://localhost:4000
export AKA_LOCAL_TOKEN=mytoken1234567890
```

Add these to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.) so they persist across sessions.

## Verifying the installation

Test each hook manually with a sample payload:

```bash
# UserPromptSubmit
echo '{"prompt":"My SSN is 123-45-6789"}' \
  | AKA_BACKEND_URL=http://localhost:4000 \
    AKA_LOCAL_TOKEN=mytoken1234567890 \
    node apps/plugin-claude-code/scripts/user-prompt-submit.js

# PreToolUse
echo '{"tool":"read_file","input":{"path":"config.env"}}' \
  | AKA_BACKEND_URL=http://localhost:4000 \
    AKA_LOCAL_TOKEN=mytoken1234567890 \
    node apps/plugin-claude-code/scripts/pre-tool-use.js
```

Both should return `{"action":"allow",...}`.

To test fail-open behaviour (backend down):

```bash
echo '{"prompt":"test"}' \
  | AKA_BACKEND_URL=http://localhost:9999 \
    AKA_LOCAL_TOKEN=mytoken1234567890 \
    node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → {"action":"allow","prompt":"test"}  ← always allows, never crashes
```

## Plugin configuration (plugin.json)

`apps/plugin-claude-code/.claude-plugin/plugin.json`:

```json
{
  "id": "aka-control-plane",
  "name": "AKA AI Control Plane",
  "version": "0.0.1",
  "description": "Intercept, scan, and govern Claude prompts and responses",
  "hooks": "./hooks/hooks.json"
}
```

## hooks.json format

```json
{
  "hooks": {
    "UserPromptSubmit": [{ "command": "node scripts/user-prompt-submit.js" }],
    "PreToolUse": [{ "command": "node scripts/pre-tool-use.js" }],
    "PostToolUse": [{ "command": "node scripts/post-tool-use.js" }]
  }
}
```

## Fail-open guarantee

The `user-prompt-submit.ts` handler wraps everything in try/catch:

```typescript
try {
  const runtime = createPluginRuntime();
  const result = await runtime.processText(payload.prompt ?? '');
  // ... write result to stdout
} catch {
  // Backend down, rule crash, anything — always allow
  process.stdout.write(JSON.stringify({ action: 'allow', prompt: payload.prompt }));
}
```

This is a hard architectural requirement. The plugin **must never** interrupt a developer's Claude session.

## Building a new adapter

To support a different AI tool (VS Code Copilot, Cursor, etc.), implement the `AkaPluginAdapter` interface from `packages/plugin-sdk`:

```typescript
interface AkaPluginAdapter {
  manifest: { id: string; tool: string; sdkVersion: string };
  capture: CaptureHooks; // beforePrompt, afterResponse, beforeToolUse, afterToolUse
  transport?: TransportOverride;
}
```

See `packages/plugin-sdk/src/types.ts` for the full interface.
