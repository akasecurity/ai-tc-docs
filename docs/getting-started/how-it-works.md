# How it works

## What it does

Every prompt, tool call, and response that flows through an agent harness session is
captured by the plugin, scanned against your rule packs, and routed to an outcome —
before the content reaches the model (prompts, tool inputs) or after it responds
(tool outputs) — then logged.

<div class="mdx-flagship__visual" style="background:#1b2531;border:1px solid rgba(0, 224, 184, 0.28);border-radius:0.8rem;margin:1.5rem 0;">
  <div class="mdx-flow__bar">
    <span>Traffic flow</span>
  </div>
  <div class="mdx-flow__body">
    <svg viewBox="0 0 420 300" role="img" aria-label="Prompts, tool calls, responses, and file reads flow into the AKA control node, which routes each one to monitor, warn, redact, block, or a manual exception.">
      <path class="mdx-flow__arrow" d="M124,25 C135,25 135,132 145,132" />
      <path class="mdx-flow__arrow" d="M124,89 C135,89 135,132 145,132" />
      <path class="mdx-flow__arrow" d="M124,153 C135,153 135,132 145,132" />
      <path class="mdx-flow__arrow" d="M124,217 C135,217 135,132 145,132" />
      <path class="mdx-flow__arrow" d="M145,132 L153,132" marker-end="url(#arch-flow-arrowhead)" />

      <rect class="mdx-flow__node" x="6" y="10" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="29">Prompts</text>
      <rect class="mdx-flow__node" x="6" y="74" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="93">Tool calls</text>
      <rect class="mdx-flow__node" x="6" y="138" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="157">Responses</text>
      <rect class="mdx-flow__node" x="6" y="202" width="118" height="30" rx="6" />
      <text class="mdx-flow__node-label" x="65" y="221">File reads</text>

      <path class="mdx-flow__arrow" d="M271,113 C290,113 290,24 300,24" marker-end="url(#arch-flow-arrowhead)" />
      <path class="mdx-flow__arrow" d="M271,124 C290,124 290,78 300,78" marker-end="url(#arch-flow-arrowhead)" />
      <path class="mdx-flow__arrow" d="M271,132 C290,132 290,132 300,132" marker-end="url(#arch-flow-arrowhead)" />
      <path class="mdx-flow__arrow" d="M271,140 C290,140 290,186 300,186" marker-end="url(#arch-flow-arrowhead)" />
      <path class="mdx-flow__arrow" d="M271,151 C290,151 290,240 300,240" marker-end="url(#arch-flow-arrowhead)" />

      <rect class="mdx-flow__control" x="161" y="105" width="110" height="54" rx="8" />
      <text class="mdx-flow__control-label" x="216" y="128">AKA</text>
      <text class="mdx-flow__control-sub" x="216" y="145">policy engine</text>

      <rect class="mdx-flow__outcome mdx-flow__outcome--monitor" x="308" y="10" width="104" height="28" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--monitor" x="360" y="28">Monitor</text>
      <rect class="mdx-flow__outcome mdx-flow__outcome--warn" x="308" y="64" width="104" height="28" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--warn" x="360" y="82">Warn</text>
      <rect class="mdx-flow__outcome mdx-flow__outcome--redact" x="308" y="118" width="104" height="28" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--redact" x="360" y="136">Redact</text>
      <rect class="mdx-flow__outcome mdx-flow__outcome--block" x="308" y="172" width="104" height="28" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--block" x="360" y="190">Block</text>
      <rect class="mdx-flow__outcome mdx-flow__outcome--exception" x="308" y="226" width="104" height="28" rx="6" />
      <text class="mdx-flow__outcome-label mdx-flow__outcome-label--exception" x="360" y="244">Exception</text>

      <defs>
        <marker id="arch-flow-arrowhead" markerWidth="8" markerHeight="8" refX="0" refY="4" orient="auto">
          <path class="mdx-flow__arrowhead" d="M0,0 L8,4 L0,8 Z" />
        </marker>
      </defs>
    </svg>
  </div>
</div>

## How rules are applied

Every event is scanned against your installed rule packs. 

* Most rules match with **regex** — a pattern shaped like an AWS access key, an email address, a bank routing number — since that covers the majority of secrets and PII, which have a predictable shape. 
* A smaller set of rules match on **keyword** presence (contextual phrases like "[REDACTED:PII] is"), and some regex matches pass through a **post-match validator** — a Luhn checksum for card numbers, a Shannon-entropy check for high-entropy secrets — to cut down false positives.
* Either way, a match becomes a **Finding**: the rule id, category, severity, and matched span, before policy decides what happens to it. 

!!! Tip
    See [Built-in Detections](../rules/built-in.md) for the full catalog, or [Writing Rules](../rules/writing-rules.md) to add your own.

## Policy outcomes

Every finding resolves to one of five outcomes:

| Outcome       | What happens                                                                 |
| ------------- | ----------------------------------------------------------------------------- |
| **Monitor**   | Logged only — every rule ships active under this default, so nothing is enforced until you promote it |
| **Warn**      | Surfaces a warning in the session; the content still goes through unchanged   |
| **Redact**    | The matched value is replaced in place before it reaches the model — only available on tool inputs/outputs, not prompt text |
| **Block**     | The prompt or tool call is stopped entirely, with a message explaining what fired |
| **Exception** | A manually-granted, exact-value override that lets one specific match through despite its rule's policy |

Promote any detection to Warn, Redact, or Block from the dashboard, per rule or per category.

## Granting exceptions

Sometimes a blocked or redacted value is legitimate such as a sandbox credential, a documented test fixture. An exception is the sanctioned way through: an explicit, audited, **exact-value** grant made from a terminal, never from inside the session it would exempt. See [Adding Exceptions](exceptions.md) for the full walkthrough.

## Key concepts

| Term          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| **Event**     | A prompt, response, or code change captured from an agent harness session |
| **Finding**   | A rule match produced by the detection engine against an event |
| **Rule**      | A JSON file describing what to detect: keyword list, regex pattern, or validator |
| **Rule Pack** | A directory of rules + mandatory fixtures with a `manifest.json` |
| **Policy**    | A decision: what action to take when a rule or category fires |
| **Plugin**    | The harness extension (hooks) that intercepts sessions — one package, used by both Claude Code and Claude Desktop |