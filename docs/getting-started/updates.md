
## Keeping up to date

The CLI and your installed plugins are versioned independently. Two commands keep
them current:

```bash
aka check-updates   # read-only: show installed vs latest for the CLI + each plugin
aka update          # update everything that's behind (asks before applying)
aka update cli      # update just the CLI
aka update claude-code   # update just one plugin
```

- `aka check-updates` changes nothing — it just reports what's available. Latest
  versions are resolved with `npm view` through your existing `~/.npmrc` auth (the
  same one that installed the packages); if the registry is unreachable, "Latest"
  shows as **unknown** and nothing is flagged.
- `aka update` shows what would change and **prompts for confirmation** before
  applying. Pass `--yes` (or `-y`) to skip the prompt (required when there's no
  interactive terminal, e.g. in a script). The CLI updates itself with
  `npm install -g @akasecurity/cli@latest`; plugins update through the `claude` plugin
  manager (**restart Claude Code afterwards** to load the new version).
- After other commands, the CLI prints a one-line **notice** when an update — or a
  newly available plugin — is waiting, with the exact command to run. It's computed
  from a once-a-day cached check (refreshed in the background, never blocking), is
  suppressed when output isn't a terminal, and can be turned off per run with
  `--no-update-check`.

!!! tip
    You can also ask your harness to update for you and it will run the CLI commands automatically.