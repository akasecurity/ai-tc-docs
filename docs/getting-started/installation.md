---
comments: true
---

# Installation

## Install the plugin

=== "Claude Code"

    The fastest path — no clone, no build. Run these in the Claude Code **terminal CLI** (`claude`):

    ```
    /plugin marketplace add akasecurity/ai-tc
    /plugin install akasecurity@ai-tc
    ```

    Restart Claude Code afterwards to load the plugin.

    [`claude-tools`](https://github.com/akasecurity/claude-tools) can also install the AKA plugin as part of setting up a Claude Code profile — see that repo for the current install command.

=== "Claude Desktop"

    Claude Desktop has no terminal, so install from the GUI instead:

    1. Open **Settings → Plugins**.
    2. **Add marketplace** and enter `akasecurity/ai-tc`.
    3. Find **aka** in the list and click **Install**.
    4. Restart Claude Desktop.

    See the [Claude Desktop plugin guide](../plugins/claude-desktop.md) for the full walkthrough.

!!! note "Coming soon"
    **Homebrew** and **npm** installation options are coming soon.

## Onboard

Inside a session, run the onboarding wizard:

```
/aka:setup
```

This asks three questions (run mode, policy, historical scan consent) and writes your preferences to `~/.aka/settings/settings.json`.

## Verify it's working

Submit a prompt containing a fake credential — the same shape a real secrets rule would match — and confirm AKA blocks it. See the fixtures in `rules/secrets/` for a concrete example payload.

You should see a block message referencing `secrets/aws-access-key`. Then run `/aka:findings` or `/aka:health` to see it recorded.

## Open the dashboard

```
/aka:dashboard
```

Navigate to `http://localhost:4319/security` to see the Events page.

## What's next

- Review the findings inside your harness with [`/aka:findings`](usage.md#akafindings)
- Use [the `aka` CLI](cli.md) for scanning, stats, and managing detection packs
- [Claude Code plugin guide](../plugins/claude-code.md) or [Claude Desktop plugin guide](../plugins/claude-desktop.md) for hook internals and configuration details
