
## Keeping up to date

The CLI and your installed plugins are versioned independently. Two commands keep
them current:

```bash
aka check-updates   # read-only: show installed vs latest for the CLI + each plugin
aka update          # update everything that's behind (asks before applying)
aka update cli      # update just the CLI
aka update claude-code   # update just one plugin
aka update cli --channel beta   # switch the CLI to another release channel
```

- `aka check-updates` changes nothing — it just reports what's available. Latest
  versions are resolved with `npm view` through your existing `~/.npmrc`
  configuration; if the registry is unreachable, "Latest" shows as **unknown** and
  nothing is flagged.
- `aka update` shows what would change and **prompts for confirmation** before
  applying. Pass `--yes` (or `-y`) to skip the prompt (required when there's no
  interactive terminal, e.g. in a script).
- **The CLI updates through whatever installed it.** An npm install runs
  `npm install -g @akasecurity/cli@latest`. A Homebrew, Scoop, or installer-script binary
  is owned by that tool, so `aka update` prints the command instead of running it:
  `brew upgrade aka`, `scoop update aka`, or re-run the installer one-liner.
- `--channel stable|beta|nightly` switches the CLI to another release channel (default: the
  channel the current copy is already on). It applies to the CLI only and is refused
  together with a plugin target.
- **Plugins update through their host's plugin manager:** Claude Code through `claude`
  (**restart Claude Code afterwards** to load the new version), Codex through
  `codex plugin add`. Antigravity installs from a local directory and has no update
  path here — `aka update` skips it. A Claude Code plugin installed by your
  organization's managed settings is refused; it updates through Claude Code's own
  autoupdate at session start, or `/plugin` → update inside a session.
- After other commands, the CLI prints a one-line **notice** when an update — or a
  newly available plugin — is waiting, with the exact command to run. It's computed
  from a once-a-day cached check (refreshed in the background, never blocking), is
  suppressed when output isn't a terminal, and can be turned off per run with
  `--no-update-check`.

!!! tip
    You can also ask your harness to update for you and it will run the CLI commands automatically.