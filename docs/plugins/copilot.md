---
comments: true
---

# GitHub Copilot Plugin

The Copilot plugin (`plugins/copilot`, npm package `@akasecurity/ai-tc-copilot`, plugin name
`aka-copilot`) hooks **GitHub Copilot** sessions. It records each session start, prompt, tool
call and tool result to the same local SQLite store (`~/.aka/data/aka.db`) as every other AKA
surface, scanning each one against your rule packs. On the Copilot CLI it can also block a shell
command before it runs.

!!! warning "Only a tool call can be stopped"
    Prompts and tool results are captured, scanned and recorded — a secret in either shows up as
    a finding — but neither is ever blocked or rewritten. AKA's notices are also usually not
    shown to you on this host. Read [Known limitations](#known-limitations) before relying on it
    for enforcement.

## Supported surfaces

One package covers three Copilot surfaces, and they are not equivalent:

| Surface            | Status                                                                                    |
| ------------------ | ----------------------------------------------------------------------------------------- |
| Copilot CLI        | Supported. Recorded on Copilot CLI 1.0.98. Windows is unconfirmed.                        |
| VS Code agent mode | **Beta.** Capture was observed (VS Code 1.141.0, Copilot Chat 0.69.0); enforcement is unverified. |
| Cloud coding agent | Not covered by a plugin install. It reads only a repository's own `.github/hooks/*.json` on the default branch. |

## Installing

There is no marketplace entry for this plugin yet: it installs from an unpacked copy of its npm
package. `aka plugins install copilot` prints these steps rather than running them.

### Copilot CLI

Run these from a folder of their own, and **keep that folder** — updates are taken from it:

```bash
mkdir -p aka-copilot && cd aka-copilot
tar -xzf "$(npm pack @akasecurity/ai-tc-copilot --silent)"
copilot plugin install ./package
copilot plugin list
aka init
```

`copilot plugin list` shows `aka-copilot` once the install has landed; the installed copy lives
under `$COPILOT_HOME/installed-plugins/_direct/` (`COPILOT_HOME` defaults to `~/.copilot`). The
CLI warns that installing from a local path is deprecated in favour of marketplace installs.

### Updating

From inside that same `aka-copilot` folder:

```bash
rm -rf package
tar -xzf "$(npm pack @akasecurity/ai-tc-copilot --silent)"
copilot plugin update aka-copilot
```

`copilot plugin update` re-copies the `package` folder into the installed plugin; it fetches
nothing itself. Remove the old `package` folder first so files the newer version deleted aren't
copied too. If the folder has been deleted, the update fails and changes nothing. `aka update`
and the CLI's update notice skip this plugin.

### VS Code agent mode (beta)

If you already installed for the Copilot CLI, reuse that `aka-copilot` folder so VS Code loads the
same copy the update steps refresh. Otherwise unpack the package the same way:

```bash
mkdir -p aka-copilot && cd aka-copilot
tar -xzf "$(npm pack @akasecurity/ai-tc-copilot --silent)"
```

Then point VS Code at the unpacked folder in your user settings:

```json
{
  "chat.plugins.enabled": true,
  "chat.useHooks": true,
  "chat.pluginLocations": { "<absolute path to the unpacked package folder>": true }
}
```

`npm pack` unpacks into a folder named `package`; use its absolute path. VS Code reads the
plugin's own `hooks.json` and expands `${PLUGIN_ROOT}` in its commands.

## Hooks

Four events are wired, each with a 30-second timeout. Only the tool call enforces:

| Event (CLI / VS Code)                       | What AKA does                                                                    |
| ------------------------------------------- | -------------------------------------------------------------------------------- |
| `sessionStart` / `SessionStart`             | opens the session and records the host, harness and project inventory            |
| `userPromptSubmitted` / `UserPromptSubmit`  | captures and scans the prompt; never blocks or rewrites it                       |
| `preToolUse` / `PreToolUse`                 | scans the tool call; can deny it, or rewrite a non-executing field               |
| `postToolUse` / `PostToolUse`               | captures and scans the tool result; never withholds it                           |

Fields scanned at `preToolUse`:

- **Copilot CLI:** `bash.command` (deny), `bash.description` (rewritten in place through
  `modifiedArgs`), and `apply_patch.input` (unverified — no payload for it was recorded).
- **VS Code:** `run_in_terminal` `command`, `explanation` and `goal` (deny only — never
  rewritten), and file-write content such as `create_file` (rewrite, unverified).

**Command text is never masked in place.** Rewriting a shell command changes what runs, so a
`redact` policy on it follows the `redactFallback` setting, which ships as `warn`: the command
runs and the only trace is a notice. Set `redactFallback` to `block` if a redact on a command has
to stop it.

A field of 32,768 characters or more is scanned in a separate thread under a deadline. For a
prompt or tool result that doesn't finish in time, AKA reports that the text went to the model
unscanned; it is never treated as clean.

## Known limitations

The plugin's [setup skill](https://github.com/akasecurity/ai-tc/blob/main/plugins/copilot/skills/setup/SKILL.md#known-limitations)
is the authority here, including a per-surface capability matrix that marks which rows were
verified against a live host. In brief:

- **Only `preToolUse` enforces.** A `block` or `redact` policy on a prompt or tool result is
  recorded as a finding and the text reaches the model unchanged.
- **Notices are usually invisible.** On the Copilot CLI in `-p` mode, AKA's stderr notices
  appeared only in the debug log (`--log-level debug --log-dir <dir>`); an interactive session
  has not been measured. In VS Code they appear only in the **GitHub Copilot Chat Hooks** output
  channel, never in the chat. Under the default `redactFallback: warn`, a warn is hard to tell
  from `monitor` on this host.
- **No setup wizard and no history backfill.** The `aka-setup` skill explains what this host can
  enforce; it does not calibrate. Use `aka init`, `aka detections`, `aka scan` and
  `aka dashboard` directly. Nothing reads sessions from before the plugin was installed.
- **On the CLI, `preToolUse` is fail-closed:** a hook that exits non-zero denies the call. AKA
  therefore writes nothing when it has no verdict, and never exits non-zero.
- **AKA never pre-approves a call.** It emits a verdict only to deny, so your own Copilot
  approval prompts are unaffected.
- **VS Code ignores matchers,** so the hook runs for every tool call, and rewrites are
  last-hook-wins: another `PreToolUse` hook's `updatedInput` can discard AKA's.
- **No forwarding line at session start.** On an attached machine, use `aka status` to see
  whether this machine forwards everything or only enrolled repositories. Copilot sign-ins are
  not recognised as work subscriptions.

This plugin has no model-judge step, so it sends nothing to a model API. Its only off-machine
path is the opt-in `aka attach`; the
[plugin README](https://github.com/akasecurity/ai-tc/blob/main/plugins/copilot/README.md) and
[SECURITY.md](https://github.com/akasecurity/ai-tc/blob/main/SECURITY.md#data-in-transit) set
out exactly what an attached machine sends.

## Uninstalling

```bash
copilot plugin uninstall aka-copilot
copilot plugin list   # aka-copilot should no longer appear
```

For VS Code, remove the folder's entry from `chat.pluginLocations`. The `aka-copilot` staging
folder is safe to delete once the plugin is removed from every surface that uses it. `~/.aka` is
untouched — see [Uninstalling](uninstalling.md) for clearing local data.
