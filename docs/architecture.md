# Architecture

## What it does

<div class="mdx-flagship__visual" style="background:#1b2531;border:1px solid rgba(0, 224, 184, 0.28);border-radius:0.8rem;margin:1.5rem 0;">
  <div class="mdx-flow__bar">
    <span>Traffic flow</span>
  </div>
  <div class="mdx-flow__body">
    <svg viewBox="0 0 420 300" role="img" aria-label="Prompts, tool calls, responses, and file reads flow into the AKA control node, which routes each one to allow, warn, redact, or block.">
      <line class="mdx-flow__arrow" x1="124" y1="25" x2="176" y2="112" marker-end="url(#arch-flow-arrowhead)" />
      <line class="mdx-flow__arrow" x1="124" y1="89" x2="176" y2="117" marker-end="url(#arch-flow-arrowhead)" />
      <line class="mdx-flow__arrow" x1="124" y1="153" x2="176" y2="125" marker-end="url(#arch-flow-arrowhead)" />
      <line class="mdx-flow__arrow" x1="124" y1="217" x2="176" y2="130" marker-end="url(#arch-flow-arrowhead)" />

      <rect class="mdx-flow__node" x="6" y="10" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="29">Prompts</text>
      <rect class="mdx-flow__node" x="6" y="74" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="93">Tool calls</text>
      <rect class="mdx-flow__node" x="6" y="138" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="157">Responses</text>
      <rect class="mdx-flow__node" x="6" y="202" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="221">File reads</text>

      <line class="mdx-flow__arrow" x1="286" y1="112" x2="308" y2="25" marker-end="url(#arch-flow-arrowhead)" />
      <line class="mdx-flow__arrow" x1="286" y1="117" x2="308" y2="89" marker-end="url(#arch-flow-arrowhead)" />
      <line class="mdx-flow__arrow" x1="286" y1="125" x2="308" y2="153" marker-end="url(#arch-flow-arrowhead)" />
      <line class="mdx-flow__arrow" x1="286" y1="130" x2="308" y2="217" marker-end="url(#arch-flow-arrowhead)" />

      <rect class="mdx-flow__control" x="176" y="94" width="110" height="54" rx="8" />
      <text class="mdx-flow__control-label" x="231" y="117">AKA</text>
      <text class="mdx-flow__control-sub" x="231" y="134">policy engine</text>

      <rect class="mdx-flow__outcome mdx-flow__outcome--allow" x="308" y="10" width="104" height="30" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--allow" x="360" y="29">Allow</text>
      <rect class="mdx-flow__outcome mdx-flow__outcome--warn" x="308" y="74" width="104" height="30" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--warn" x="360" y="93">Warn</text>
      <rect class="mdx-flow__outcome mdx-flow__outcome--redact" x="308" y="138" width="104" height="30" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--redact" x="360" y="157">Redact</text>
      <rect class="mdx-flow__outcome mdx-flow__outcome--block" x="308" y="202" width="104" height="30" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--block" x="360" y="221">Block</text>

      <defs>
        <marker id="arch-flow-arrowhead" markerWidth="8" markerHeight="8" refX="4" refY="4" orient="auto">
          <path class="mdx-flow__arrowhead" d="M0,0 L8,4 L0,8 Z" />
        </marker>
      </defs>
    </svg>
  </div>
</div>

Every prompt, tool call, and response that flows through an agent harness session is
captured by the plugin, scanned against your rule packs, and routed to an outcome —
before the content reaches the model (prompts, tool inputs) or after it responds
(tool outputs) — then logged.

## The OSS stack at a glance

The stack is a TypeScript-strict pnpm monorepo:

| Package                   | Role                                                                |
| -------------------------- | ------------------------------------------------------------------- |
| `apps/plugin-claude-code` | Hooks for Claude Code and Claude Desktop — captures prompts/responses, fail-open |
| `apps/web-ui`             | OSS Next.js dashboard — reads the local SQLite store directly       |
| `apps/cli`                | The `aka` CLI — local scan, stats, dashboard, plugin management     |
| `packages/detections`     | Pure detection engine — scan + redact, no I/O                       |
| `packages/schema`         | Zod contracts — single source of truth for all API shapes           |
| `packages/persistence`    | Local SQLite store adapter — no backend required                    |
| `packages/plugin-sdk`     | Shared adapter interface + runtime lifecycle                        |
| `rules/`                  | Detection rule packs (core-pii, secrets)                            |

See the [System Overview](architecture/overview.md) for the full picture: repository
layout and the local-first data flow from hook to store to dashboard.
