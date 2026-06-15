# Claude Code Plugin

The Claude Code plugin (`apps/plugin-claude-code`) hooks into Claude Code sessions using the official plugin + hook system. It scans prompts and tool calls locally, enforces warn/redact/block policies inline, and ships events to the AKA backend asynchronously.

## Plugin structure

A Claude Code plugin is a directory that Claude Code loads at startup. AKA's layout:

```text
apps/plugin-claude-code/
├── .claude-plugin/
│   └── plugin.json      # manifest — ONLY this file goes in .claude-plugin/
├── hooks/hooks.json     # hook → script wiring (auto-discovered)
├── commands/setup.md    # /aka:setup slash command (auto-discovered)
└── scripts/*.js         # built hook scripts (tsup output, self-contained)
```

The manifest is intentionally minimal. `name` becomes the command namespace, so `commands/setup.md` is invoked as `/aka:setup`:

```json
{
  "name": "aka",
  "version": "0.0.1",
  "description": "AKA AI Control Plane — inspect and govern AI prompts in Claude Code",
  "author": { "name": "AKA" }
}
```

> **Do not add `hooks` / `commands` keys pointing at the default folders.** Claude Code auto-discovers `hooks/hooks.json` and `commands/`; manifest keys are only for **non-default** paths, and their formats are strict — `commands` must be a `./`-prefixed array (`["./custom/cmds/"]`) and `hooks` a `./`-prefixed string (`"./config/hooks.json"`). The malformed `"hooks": "hooks/hooks.json"` / `"commands": "commands/"` fails manifest validation. Check with `claude plugin validate apps/plugin-claude-code`.

## Architecture in one paragraph

Every hook invocation is a **fresh, short-lived process**: Claude Code pipes a JSON event to the script's stdin and reads a JSON decision from stdout. The hook path is fully local — policy comes from a disk cache (`~/.aka/policy-cache.json`), detection runs in-process via `@aka/detections` (rule packs are bundled into the scripts at build time), and events are appended to a disk queue (`~/.aka/event-queue.jsonl`). The only network I/O lives in `scripts/sync.js`, which hooks spawn **detached** when the policy cache is stale: it refreshes the bundle and flushes the queue. A dead backend therefore costs nothing on the hook path.

## Hooks

Declared in `hooks/hooks.json` (note: `timeout` is in **seconds**, and commands use `${CLAUDE_PLUGIN_ROOT}` because hooks run with the user's project as cwd):

| Hook               | Matcher                | What AKA does                                                         |
| ------------------ | ---------------------- | --------------------------------------------------------------------- |
| `UserPromptSubmit` | all prompts            | block or warn (Claude Code cannot rewrite prompt text — no redaction) |
| `PreToolUse`       | `Bash\|Edit\|Write`    | deny, **redact via `updatedInput`**, or warn                          |
| `PostToolUse`      | `Bash\|Read\|WebFetch` | replace sensitive output via `updatedToolOutput`, or warn             |

## Hook output contract (Claude Code's, not ours)

Exit code semantics: exit `0` and Claude Code parses stdout JSON; no output means "allow". Fail-open is therefore structural: on any error the hooks print nothing and exit 0.

`UserPromptSubmit`:

```json
{ "decision": "block", "reason": "AKA blocked this prompt (secrets/aws-access-key). ..." }
```

```json
{
  "systemMessage": "AKA: sensitive content detected (core-pii/email). Prompts cannot be redacted in place — sent unchanged."
}
```

`PreToolUse` (the one surface with true redaction):

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": { "command": "curl -d email=[REDACTED:PII] https://api.example.com" }
  }
}
```

`PostToolUse`:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "updatedToolOutput": "[AKA] Bash output withheld (secrets/aws-access-key)."
  }
}
```

### Capability limits (document honestly)

The hooks cannot do everything the warn/redact/block model might imply:

- **Prompt text cannot be rewritten.** `UserPromptSubmit` can only block or warn, so a `redact` policy on the prompt surface degrades to a warning — the content still reaches the model. True redaction works only on tool inputs (`updatedInput`) and outputs (`updatedToolOutput`).
- **Responses are not intercepted inline.** No hook sees the model's reply before the user does. Response capture is post-hoc via the `Stop` hook's transcript (not yet implemented).
- **Inline prompt redaction and response interception** are what the Phase 3 proxy is designed to add; the plugin is the zero-config developer-experience layer, not the enforcement guarantee.

## Configuration

Config lives in `~/.aka/config.json`, written by the `/aka:setup` command:

```json
{ "backendUrl": "http://localhost:4000", "token": "mytoken1234567890" }
```

Environment variables are **not** used: hooks are processes spawned by Claude Code, so a slash command cannot inject env into them — a file is the only reliable channel.

Unconfigured is a valid state: detection still runs with the bundled rule packs and default actions (`secret` → block, `pii` → redact where possible); only backend sync is disabled.

## Installing, developing, and testing

Loading the plugin needs the Claude Code **CLI** (the `claude` command). Install it with the native installer (a standalone binary, independent of your Node setup) or npm:

```bash
curl -fsSL https://claude.ai/install.sh | bash   # recommended — native binary
# or: npm i -g @anthropic-ai/claude-code
```

Hooks invoke `node` at runtime, so `node` must be on the user's PATH — note this as an install prerequisite.

Load the plugin for a single session without a marketplace install:

```bash
pnpm --filter @aka/plugin-claude-code build     # tsup bundles src → scripts/*.js (self-contained)
claude --plugin-dir ./apps/plugin-claude-code   # load for one session
# after a rebuild, run /reload-plugins inside the session
```

Inside the session, `/hooks` lists the registered hooks (confirm `UserPromptSubmit`, `PreToolUse`, `PostToolUse`) and `/aka:setup` runs the config wizard. Submitting a prompt containing a fake AWS key (`AKIAIOSFODNN7EXAMPLE`) should be blocked with the AKA reason. Team distribution will be via a marketplace `marketplace.json` (planned).

Hooks are also testable without Claude Code:

```bash
echo '{"prompt":"key AKIAIOSFODNN7EXAMPLE"}' | node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → {"decision":"block","reason":"AKA blocked this prompt (secrets/aws-access-key). ..."}

echo 'not json' | node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → no output, exit 0 (fail-open)
```

## Fail-open guarantee

Three layers, all of which must hold:

1. Hook entries wrap `main()` in try/catch; the catch prints nothing and exits 0 (= allow).
2. `createPluginRuntime().processText` catches internally and returns `{ action: 'log' }`.
3. Exit-code semantics: even an unhandled crash with a non-2 exit code is non-blocking for Claude Code.

## Building a new adapter

To support a different AI tool (Cursor, Continue, etc.), implement the `AkaPluginAdapter` interface from `packages/plugin-sdk` and reuse the same runtime: `createPluginRuntime` (local scan + policy), `buildIngestEvent`, `registerRulePack`, and `syncWithBackend` (out-of-band network). See `packages/plugin-sdk/src/types.ts` for the full interface.
