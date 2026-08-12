---
comments: true
---

# Antigravity Plugin

The Antigravity plugin (`plugins/antigravity`) hooks Google **Antigravity** CLI (`agy`) sessions.
It writes to the same local SQLite store (`~/.aka/data/aka.db`) as every other AKA surface, so
findings from an Antigravity session show up in `aka findings`, the dashboard, and the other
plugins' read commands.

!!! warning "This host is the narrowest of the three"
    Antigravity's hook contract cannot do several things Claude Code's can — prompts cannot be
    blocked or redacted, tool results are not scanned live, and hooks only fire in the **CLI**,
    not the IDE. Read [Known limitations](#known-limitations) before relying on it for
    enforcement.

## Installing

Antigravity has **no plugin marketplace**: `agy plugin install` takes a local directory path
rather than a `<plugin>@<marketplace>` ref. So the install is a fetch-then-point:

```bash
npm pack @akasecurity/ai-tc-antigravity
tar -xzf akasecurity-ai-tc-antigravity-*.tgz
agy plugin install ./package
```

Then scaffold the local store if you haven't already:

```bash
aka init
```

`aka plugins install antigravity` prints exactly these steps rather than running them — there are
no marketplace coordinates for the CLI to drive, and `aka update` skips the plugin for the same
reason. Re-run the `npm pack` sequence to move to a newer version.

`node` must be on your PATH; the hooks are Node scripts that `agy` spawns.

## Hooks

Antigravity exposes five events — `PreInvocation`, `PostInvocation`, `PreToolUse`, `PostToolUse`
and `Stop`. There is **no `SessionStart` and no `UserPromptSubmit`**, so the adapter maps what it
can:

| Hook            | Matcher      | What AKA does                                                                                  |
| --------------- | ------------ | ---------------------------------------------------------------------------------------------- |
| `PreInvocation` | all          | stands in for `SessionStart` — inventory pass on the first invocation of a conversation (claimed internally, so later invocations no-op); injects no steps |
| `PreToolUse`    | `.*`         | deny or warn on shell commands and file writes                                                   |
| `PostToolUse`   | `.*`         | records the tool call (the payload carries **no tool output** to scan)                           |
| `Stop`          | end of loop  | triggers the background reconcile worker; never emits `{"decision":"continue"}`                  |

Antigravity is a **fail-closed** host: a hook that prints nothing is treated as a *deny*. AKA's
hooks therefore always print an explicit allow (`{}`) — on success, on bad input, on a crash, and
on their own 8-second watchdog.

## Skills

Like Codex, the read surface ships as skills (`aka-setup`, `aka-findings`, `aka-health`,
`aka-recommend`, `aka-audit`, `aka-scan`, `aka-tokens`, `aka-detections`, `aka-exceptions`,
`aka-dashboard`) rather than `/aka:*` slash commands. They load in the Antigravity **IDE** as well
as the CLI, and read the same local store either way.

## Known limitations

These are contract limits of the host, not bugs — none of them are worked around, and none should
be described as covered:

- **The IDE does not run plugin hooks.** Skills and rules load in the Antigravity IDE and appear
  under Customizations, but `hooks.json` entries appear never to fire there — hooks execute in the
  `agy` CLI. (Google documents this neither way; treat it as observed behaviour.) So live
  detection and blocking apply to **CLI sessions only**. IDE work is covered after the fact by
  `aka scan` and the dashboard.
- **Prompts cannot be blocked or redacted.** No event on this host hands a hook the user's message
  before the model sees it — `PreInvocation` carries `invocationNum`, `initialNumSteps` and the
  common fields, but no prompt text, and its output can only *add* steps. A secret pasted into a
  prompt reaches the model; AKA records it (so it appears in findings and the dashboard) but
  cannot stop it. This is the single biggest difference from the Claude Code plugin.
- **A `redact` policy denies instead of masking.** `PreToolUse`'s output is
  `{ decision, reason, permissionOverrides }` with no field for modified arguments, so AKA denies
  the tool call and explains what to remove. This applies to file writes as well as shell commands.
- **A `warn` is recorded but not shown inline.** `PreToolUse` has no message channel (its `reason`
  accompanies a deny), so warns are ledgered silently — see them via `aka-findings` or the
  dashboard.
- **Tool results are not scanned live.** The `PostToolUse` payload does not include the tool's
  output at all, so secrets in command output are caught after the fact, not before the model
  sees them.
- **Token usage produces nothing.** Antigravity's transcript carries no token, model, or cost
  field, so usage reporting stays empty for Antigravity sessions.
- **`multi_replace_file_content` is not scanned.** Its arguments nest edits in an undocumented
  container, so AKA does not guess at the shape. `write_to_file`, `replace_file_content`, and
  `run_command` are scanned.
- **Historical scanning reads transcripts, not tool arguments.** Past prompts and model messages
  *are* found by the backfill; a secret sitting in a past `run_command` line or `write_to_file`
  body is not (the live `PreToolUse` hook does scan those, so this gap is about past sessions).

## Onboarding and the model-judge egress

`aka-setup` runs the same calibration wizard with the same two consents, but this host's egress is
weaker than Claude Code's and Codex's in three named ways, all of which the wizard states before
asking for consent:

- **The `agy` CLI documents no ephemeral mode.** Every run is written to the conversation store
  under `~/.gemini/antigravity/brain/` — raw values and all — so AKA deletes the judge's own
  conversation itself when the run ends. That deletion is best-effort: a process killed between
  the write and the cleanup leaves the conversation on disk.
- **The prompt travels on the command line**, because `agy` documents no way to read a prompt from
  stdin. For the life of each run, raw values are visible to anything that can list processes on
  the machine.
- **Deleting the conversation is not network isolation.** It is a local-write cleanup; it cannot
  recall what was already sent.

Declining the model-judge consent leaves AKA on the conservative severity floor. See [Why the
setup scan sends findings to the
model](../getting-started/installation.md#why-the-setup-scan-sends-findings-to-the-model).

## Uninstalling

Remove the plugin with `agy`'s own plugin manager, then restart the CLI. See
[Uninstalling](uninstalling.md) for what remains on disk afterwards.
