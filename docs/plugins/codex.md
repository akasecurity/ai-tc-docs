---
comments: true
---

# Codex CLI Plugin

The Codex plugin (`plugins/codex`) hooks OpenAI **Codex CLI** sessions through Codex's own
plugin + hook system. Like every AKA plugin it is an independent peer: detection runs
in-process, findings land in the same local SQLite store (`~/.aka/data/aka.db`) the Claude Code
plugin, the `aka` CLI, and the dashboard all read, and no backend is required.

## Installing

Codex CLI installs plugins from a marketplace, the same `<bin> plugin …` surface Claude Code
exposes. AKA's Codex plugin is served from this repo's own marketplace file:

```bash
codex plugin marketplace add akasecurity/ai-tc
codex plugin add aka-codex@ai-tc
```

Codex has no `plugin install` verb — `add` installs and also updates. Restart Codex afterwards to load the plugin. `node` must be on your PATH — the hooks are Node
scripts that Codex spawns.

!!! note "Why a different marketplace than Claude Code"
    The canonical `akasecurity/marketplace` repo currently carries the Claude Code plugin only,
    so Codex resolves `akasecurity/ai-tc` instead. The plugin name differs too (`aka-codex`, not
    `ai-tc`) — the two refs are deliberately distinct so neither host's install ledger can be
    mistaken for the other's.

If you have the [`aka` CLI](../getting-started/cli.md), it drives the same two commands for you:

```bash
aka plugins install codex
aka update codex          # later, to update it
```

## Hooks

Declared in `hooks/hooks.json` (`timeout` is in seconds; commands resolve `${PLUGIN_ROOT}`):

| Hook               | Matcher            | What AKA does                                                                    |
| ------------------ | ------------------ | -------------------------------------------------------------------------------- |
| `SessionStart`     | all sessions       | resolve and upsert this machine's inventory, open the session's audit-event root; on an attached machine, say where the session's activity goes |
| `UserPromptSubmit` | all prompts        | block or warn                                                                     |
| `PreToolUse`       | `Bash\|apply_patch` | deny or warn; **redact in place** for `apply_patch` content only (see below)       |
| `PostToolUse`      | `Bash\|apply_patch\|webrun\|mcp__.*` | withhold the whole tool result, or warn                          |
| `Stop`             | end of turn        | trigger the background token-usage reconcile worker (detached; adds no latency)   |

Codex's event names and stdin shapes match Claude Code's, which is why the two adapters share a
runtime. The **outputs** differ in two ways that matter:

- **A `redact` policy on a prompt blocks.** Codex cannot rewrite prompt text on
  `UserPromptSubmit` either, and warning-and-passing would send the raw secret to the model, so
  the decision escalates to `{"decision":"block"}` — the prompt is rejected, the reason is shown,
  and the session continues so you can resubmit.
- **Tool output cannot be spliced.** Codex's `PostToolUse` has no `updatedToolOutput` field. The
  strongest available action is `{"decision":"block","reason":"…"}`, which replaces the **entire**
  tool result with the reason and continues the model from there — so a `redact` outcome on a
  response escalates to a whole-result withhold rather than a partial mask.

- **A shell command is never masked in place.** Rewriting it would silently change what runs, so
  a `redact` policy on a `Bash` command follows the `redactFallback` setting in
  `~/.aka/settings/settings.json`. It ships as **`warn`**: the command runs with the value
  unmasked and AKA says so in the session. Set it to `block` to deny the call instead.
  `apply_patch` content is stored text, not a command, so it is always masked in place.

Live scanning covers `Bash` commands and `apply_patch` file writes before they run, and the
output of shell calls, Codex's web tool and MCP tools after they return — including the tool
calls a code-mode `exec` makes (confirmed on codex-cli 0.160.0). Web and MCP tool **inputs** are
not scanned on the way in. A very large tool result is scanned up to a bound (5 million
characters or 2,000 pieces, and about 7 seconds); the rest reaches the model unscanned, and a
cut-short scan counts as a fail-open in `aka status`.

!!! note "Older Codex releases"
    On a Codex release that fires these hooks for `Bash` alone, `apply_patch` content is not
    scanned at all — live or after the fact. Only the changed paths are recorded.

The full text of every `apply_patch` call is recorded in the local store, with detected values
masked. Shell commands and tool output are recorded only when something is detected.

The reversible secret vault is not wired for Codex sessions: everything AKA redacts here is
one-way, and a vault pointer (`[[aka:...]]`) inside a `Bash` command is denied. When vault consent
was granted by another plugin on the machine, `SessionStart` still injects a brief telling the
model what a pointer means.

Fail-open is structural, exactly as in Claude Code: on any error the hooks print nothing and
exit 0, which Codex reads as "allow".

## Skills, not slash commands

Codex has no `/aka:*` command namespace, so the read surface ships as **skills** under
`plugins/codex/skills/` — one per command, with the same output as the Claude Code equivalents:

| Skill             | Shows                                                                           |
| ----------------- | ------------------------------------------------------------------------------- |
| `aka-setup`       | the onboarding / calibration wizard                                             |
| `aka-dashboard`   | launches the local web dashboard via the `aka` CLI                              |
| `aka-health`      | detection activity, action breakdown, category coverage                         |
| `aka-findings`    | recent findings with the **masked** match                                       |
| `aka-recommend`   | findings grouped by category, plus next steps                                   |
| `aka-audit`       | the decision log                                                                |
| `aka-scan`        | scan working-tree source files for insecure code patterns                       |
| `aka-tokens`      | token usage per provider/model, plus estimated cost                             |
| `aka-detections`  | installed detection packs — version, rules, enabled, update available           |
| `aka-exceptions`  | active detection exceptions (read-only)                                         |

`aka-exceptions` and `aka-detections` are read-only for the same reason they are in Claude Code:
the session being policed must not be able to grant its own bypass or silently change what runs.
Creating an exception happens out-of-band via `aka exception approve`; updating packs via
`aka detections update` or the dashboard's **Update** button.

## Onboarding and the model-judge egress

`aka-setup` is the same calibration wizard, with the same two separate consents (historical
access, then model-judge consent) and the same settings file at `~/.aka/settings/settings.json`.
The one host-specific detail: the judgment step reaches the model through **`codex` CLI
subprocesses** rather than `claude -p`. Those run ephemerally, so raw values are never written
into `~/.codex/sessions` — but that is *local-write* isolation, **not** network isolation. A copy
of each judged value leaves the machine, like any other Codex prompt.

See [Why the setup scan sends findings to the
model](../getting-started/installation.md#why-the-setup-scan-sends-findings-to-the-model) for the
full payload description and the revocation limits.

## Uninstalling

```bash
codex plugin remove aka-codex@ai-tc
```

See [Uninstalling](uninstalling.md) for what that does and doesn't clean up.
