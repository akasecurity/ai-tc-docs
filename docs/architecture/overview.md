# System Overview

## Key concepts

| Term          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| **Event**     | A prompt, response, or code change captured from an agent harness session |
| **Finding**   | A rule match produced by the detection engine against an event |
| **Rule**      | A JSON file describing what to detect: keyword list, regex pattern, or validator |
| **Rule Pack** | A directory of rules + mandatory fixtures with a `manifest.json` |
| **Policy**    | A decision: what action to take when a rule or category fires |
| **Plugin**    | The harness extension (hooks) that intercepts sessions — one package, used by both Claude Code and Claude Desktop |

## Repository layout

```
ai-tc/
├── apps/
│   ├── docs/                This MkDocs site
│   └── plugin-claude-code/  Hooks for Claude Code and Claude Desktop
├── packages/
│   ├── schema/           Zod contracts — single source of truth
│   ├── config/           Env loader — sole reader of process.env
│   ├── detections/       Pure detection engine (no I/O) — runs identically
│   │                     everywhere detection happens (plugin, backend)
│   ├── client/           Generated API client (sole caller of fetch)
│   ├── plugin-sdk/       AkaPluginAdapter interface + runtime
│   └── ui-kit/           Shared React component library (stub)
├── rules/
│   ├── core-pii/         Email + SSN rules with fixtures
│   └── secrets/          AWS key + GitHub PAT rules with fixtures
├── tools/
│   ├── migrator/         CLI for applying DB migrations
│   └── rules-publisher/  CLI that publishes rules/ packs to the registry
└── skills/
    ├── backend-conventions/SKILL.md
    └── write-detection-rule/SKILL.md
```

This is the open-source surface: the plugin, the CLI, the local SQLite store, and
the OSS web-ui. AKA also has a separate enterprise control plane for teams — see
[OSS vs. Enterprise](../oss-vs-enterprise.md) for how the two relate; it isn't
covered further here.

## How data flows

The plugin is fully useful with **zero backend**. It depends on a single
**`DataGateway`** port (defined in `@akasecurity/plugin-sdk`); `@akasecurity/plugin-runtime`
resolves the implementation from the run mode:

- **standalone** (default) — a SQLite store at `~/.aka/data/aka.db` via
  `@akasecurity/persistence`, read directly by the OSS web-ui and CLI.
- **attached** — the enterprise backend's HTTP API via `@akasecurity/client` (see
  [OSS vs. Enterprise](../oss-vs-enterprise.md)).

Detection runs in-process in either mode, and results surface through slash
commands (`/aka:health`, `/aka:findings`, `/aka:recommend`, `/aka:audit`) or the
dashboard. The gateway and shared data shapes live in `@akasecurity/plugin-sdk` /
`@akasecurity/persistence`, so a new harness adapter adds only a thin tool-specific
layer, never a copy of the storage/detection logic.

<div class="mdx-diagram" style="margin:1.5rem 0;">
  <div class="mdx-terminal__bar mdx-diagram__bar">
    <span>Local architecture</span>
  </div>
  <div class="mdx-diagram__body">
    <svg viewBox="0 0 380 340" role="img" aria-label="Claude Code or Claude Desktop talks to the AKA plugin, which writes to a local SQLite store that the dashboard and CLI read directly, all inside your machine with no backend.">
      <rect class="mdx-diagram__boundary" x="10" y="34" width="360" height="296" rx="14" />
      <text class="mdx-diagram__boundary-label" x="28" y="24">your machine &middot; no backend</text>

      <rect class="mdx-diagram__node" x="50" y="52" width="280" height="46" rx="8" />
      <text class="mdx-diagram__node-label" x="190" y="80">Claude Code / Claude Desktop</text>

      <line class="mdx-diagram__arrow" x1="190" y1="98" x2="190" y2="130" marker-end="url(#overview-diagram-arrowhead)" />
      <text class="mdx-diagram__arrow-label" x="204" y="118">hooks</text>

      <rect class="mdx-diagram__node" x="50" y="132" width="280" height="46" rx="8" />
      <text class="mdx-diagram__node-label" x="190" y="160">AKA plugin</text>

      <line class="mdx-diagram__arrow" x1="190" y1="178" x2="190" y2="210" marker-end="url(#overview-diagram-arrowhead)" />
      <text class="mdx-diagram__arrow-label" x="204" y="198">writes</text>

      <rect class="mdx-diagram__node mdx-diagram__node--accent" x="50" y="212" width="280" height="46" rx="8" />
      <text class="mdx-diagram__node-label mdx-diagram__node-label--accent" x="190" y="240">~/.aka/data/aka.db</text>

      <line class="mdx-diagram__arrow" x1="190" y1="258" x2="190" y2="290" marker-end="url(#overview-diagram-arrowhead)" />
      <text class="mdx-diagram__arrow-label" x="204" y="278">reads</text>

      <rect class="mdx-diagram__node" x="50" y="292" width="280" height="46" rx="8" />
      <text class="mdx-diagram__node-label" x="190" y="320">Dashboard / CLI</text>

      <defs>
        <marker id="overview-diagram-arrowhead" markerWidth="8" markerHeight="8" refX="4" refY="4" orient="auto">
          <path class="mdx-diagram__arrowhead" d="M0,0 L8,4 L0,8 Z" />
        </marker>
      </defs>
    </svg>
  </div>
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
[Claude Code plugin guide](../plugin/claude-code.md#hooks) for the exact hook-by-hook
contract.

In `attached` mode the same event is additionally queued to the enterprise backend
over HTTP, and the plugin periodically refreshes a cached policy bundle from it —
see [OSS vs. Enterprise](../oss-vs-enterprise.md) for that side of the flow.

## Dashboard

The OSS **web-ui** (`apps/web-ui`, Next.js) reads the local SQLite store directly
through `@akasecurity/persistence` — no backend, no HTTP. It shares its view
components with the enterprise dashboard, so the same Findings, Detections, Data
Shares, and Inventory pages work identically over local data. Launch it with
`aka dashboard` — see the [CLI](../getting-started/cli.md) and
[Local Mode](../deployment/local.md).
