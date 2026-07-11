# Claude Desktop Plugin

Claude Desktop runs the **same plugin package** as Claude Code — `apps/plugin-claude-code`, identical hooks (`SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`), the same fail-open guarantee, the same local SQLite store (`~/.aka/data/aka.db`), and the same `/aka:*` commands. See the [Claude Code plugin guide](claude-code.md) for the full architecture, hook contract, and command reference — this page only covers what's different about running it inside Desktop.

## Installing

Claude Desktop has no terminal, so plugin management happens in **Settings → Plugins** instead of the `/plugin` slash commands used in the Code CLI:

1. Open **Settings → Plugins**.
2. **Add marketplace** and enter the marketplace repo: `akasecurity/ai-tc`.
3. Find **aka** in the installed marketplace's plugin list and click **Install**.
4. Restart Claude Desktop to load the plugin.

> **Pre-release — private GitHub Packages.** Same requirement as Claude Code: while AKA is pre-release, the marketplace resolves an npm source hosted on private GitHub Packages, so Desktop's install step still needs a `~/.npmrc` on the machine pointing the `@akasecurity` scope at `npm.pkg.github.com` with a `read:packages` token. See [Claude Code → Distribution](claude-code.md#distribution-marketplace-npm) for the exact `.npmrc` lines — write it once before clicking Install.

## Onboarding and everyday use

Once installed, `/aka:setup` and every other `/aka:*` command work exactly as documented for Claude Code — type them into the Desktop chat box the same way. Findings, policy, and the local store are shared: `~/.aka/data/aka.db` is the same file regardless of whether a session ran in Code or Desktop, so `/aka:findings` in one shows detections from the other.

## What's Code-CLI-only

The following depend on a terminal and don't apply to Desktop:

- Loading an unpublished build for a single session (`claude --plugin-dir ...`)
- `/reload-plugins` after a local rebuild
- Testing hook scripts directly via stdin (`echo '...' | node .../user-prompt-submit.js`)

For that contributor/dev workflow, see [Installing, developing, and testing](claude-code.md#installing-developing-and-testing) in the Claude Code guide — building from source and loading the plugin for development is done through the Code CLI even if you'll also use it in Desktop.
