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
      <circle class="mdx-diagram__pulse" r="4">
        <animateMotion dur="2.4s" begin="0s" repeatCount="indefinite" path="M190,98 L190,130" />
      </circle>

      <rect class="mdx-diagram__node" x="50" y="132" width="280" height="46" rx="8" />
      <text class="mdx-diagram__node-label" x="190" y="160">AKA plugin</text>

      <line class="mdx-diagram__arrow" x1="190" y1="178" x2="190" y2="210" marker-end="url(#overview-diagram-arrowhead)" />
      <text class="mdx-diagram__arrow-label" x="204" y="198">writes</text>
      <circle class="mdx-diagram__pulse" r="4">
        <animateMotion dur="2.4s" begin="0.8s" repeatCount="indefinite" path="M190,178 L190,210" />
      </circle>

      <rect class="mdx-diagram__node mdx-diagram__node--accent" x="50" y="212" width="280" height="46" rx="8" />
      <text class="mdx-diagram__node-label mdx-diagram__node-label--accent" x="190" y="240">~/.aka/data/aka.db</text>

      <line class="mdx-diagram__arrow" x1="190" y1="258" x2="190" y2="290" marker-end="url(#overview-diagram-arrowhead)" />
      <text class="mdx-diagram__arrow-label" x="204" y="278">reads</text>
      <circle class="mdx-diagram__pulse" r="4">
        <animateMotion dur="2.4s" begin="1.6s" repeatCount="indefinite" path="M190,258 L190,290" />
      </circle>

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
[Claude Code plugin guide](../plugins/claude-code.md#hooks) for the exact hook-by-hook
contract.

## Dashboard

The OSS **web-ui** (`apps/web-ui`, Next.js) reads the local SQLite store directly
through `@akasecurity/persistence` — no backend, no HTTP. It shares its view
components with the enterprise dashboard, so the same Findings, Detections, Data
Shares, and Inventory pages work identically over local data. Launch it with
`aka dashboard` — see the [CLI](../getting-started/cli.md).


