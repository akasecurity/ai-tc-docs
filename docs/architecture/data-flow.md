# Data Flow

## Local flow

The plugin is fully useful with **zero backend**. It depends on a single
**`DataGateway`** port (defined in `@akasecurity/plugin-sdk`), implemented by
`@akasecurity/plugin-runtime` as a local SQLite store at `~/.aka/data/aka.db` via
`@akasecurity/persistence`, read directly by the OSS web-ui and CLI.

Detection runs in-process, and results surface through slash
commands (`/aka:health`, `/aka:findings`, `/aka:recommend`, `/aka:audit`) or the
dashboard. The gateway and shared data shapes live in `@akasecurity/plugin-sdk` /
`@akasecurity/persistence`, so a new harness adapter adds only a thin tool-specific
layer, never a copy of the storage/detection logic.

<div class="mdx-diagram-wrap" style="margin:1.5rem 0;">
--8<-- "overrides/partials/local_arch_diagram.html"
</div>

## Event ingestion, step by step

Take a single prompt in a Claude session:

```
1. User types a prompt
2. UserPromptSubmit hook fires
     → user-prompt-submit.js reads stdin
     → createPluginRuntime().processText()
         → scan() against loaded rules
         → resolve policy action
         → redact if action = redact
         → writeEvent() to the local store
     → writes {action, prompt?} to stdout
3. Claude receives the (possibly redacted) prompt and responds
```

The same path runs for tool calls (`PreToolUse` → possible `updatedInput`) and tool
outputs (`PostToolUse` → possible `updatedToolOutput`). See the
[Claude Code plugin guide](../plugins/claude-code.md#hooks) for the exact hook-by-hook
contract.

## Dashboard

The OSS **web-ui** (`apps/web-ui`, Next.js) reads the local SQLite store directly
through `@akasecurity/persistence` — no backend, no HTTP. It shares its view
components with the enterprise dashboard, so the same Findings, Detections, Data
Shares, and Inventory pages work identically over local data. Launch it with
`aka dashboard` — see the [CLI](../getting-started/cli.md).


