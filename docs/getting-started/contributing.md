
## Build from source (contributors)

Building AKA from source instead of installing the published plugin — useful if you're developing AKA itself. This workflow uses the Claude Code **terminal CLI**; there's no equivalent for Desktop, since it depends on shell access.

**Prerequisites:** Node.js 24+ (the repo's `.nvmrc` pins `24`) and pnpm 10+, Claude Code installed and signed in.

### 1. Clone and install

```bash
git clone https://github.com/akasecurity/ai-tc
cd ai-tc
pnpm run setup   # pnpm install + lefthook hooks
pnpm --filter @akasecurity/cli build
```

`pnpm run setup`, not `pnpm setup`: `setup` is also a built-in pnpm command (it configures
`PNPM_HOME` in your shell profile), and a built-in wins over a package script of the same name.

### 2. Set up your local AKA home

```bash
pnpm --filter @akasecurity/cli exec aka init
```

This creates `~/.aka/settings/settings.json` (your preferences) and
`~/.aka/data/aka.db` (the local SQLite store), seeded with default detection
policies.

### 3. Load the plugin for a session

```bash
pnpm turbo run build --filter=@akasecurity/ai-tc-claude-code
claude --plugin-dir ./plugins/claude-code
```

### 4. Test the plugin hook

Run the hook script directly with a sample payload — a fake credential in the
same shape a real secrets rule would match — to verify it works:

```bash
echo '{"prompt":"here is a test credential: see rules/secrets fixtures for the exact shape"}' \
  | node plugins/claude-code/scripts/user-prompt-submit.js
# → {"decision":"block","reason":"AKA blocked this prompt — flagged secrets/aws-access-key ..."}
```

For a concrete example payload, see the fixtures in `rules/secrets/`.

### 5. Open the dashboard

```bash
pnpm --filter @akasecurity/cli exec aka dashboard
```

From a source checkout, the first run builds the web UI, which can take a minute. Navigate to
`http://localhost:4319/security` to see the Security page. The finding from step 4 should appear
once you've triggered it from inside a real Claude Code session (the direct hook invocation
above only tests the script).

## Before you open a PR

The root scripts run across the workspace through Turborepo:

```bash
pnpm typecheck      # turbo typecheck + the root tsconfig
pnpm lint           # turbo lint + root lint + the portability gate
pnpm test           # turbo test
pnpm format:check   # prettier --check .
pnpm build          # turbo build
pnpm bench          # benchmarks (advisory only)
```

`pnpm typecheck`, `pnpm lint`, `pnpm test` and `pnpm format:check` must pass. The git hooks
`pnpm run setup` installs (lefthook) do part of this for you:

| Hook         | Runs                                                                 |
| ------------ | -------------------------------------------------------------------- |
| `pre-commit` | `prettier --write` on staged files, then re-stages them              |
| `commit-msg` | `commitlint` — commit messages must follow [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`) |
| `pre-push`   | `pnpm lint`                                                          |

The repo's [CONTRIBUTING.md](https://github.com/akasecurity/ai-tc/blob/main/CONTRIBUTING.md) is
the full reference — required CI checks, the dependency-audit and waiver policy, and how to build
the browser extension (`pnpm --filter @akasecurity/plugin-browser-extension build`).
