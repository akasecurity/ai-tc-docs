---
comments: true
---

# Claude Code Plugin

The Claude Code plugin (`apps/plugin-claude-code`) hooks into Claude Code sessions using the official plugin + hook system. It scans prompts and tool calls locally and enforces warn/redact/block policies inline — writing findings to a local SQLite store by default, with zero backend required.

!!! note 
    **Same plugin, two interfaces.** This is the same package that runs inside Claude Desktop — identical hooks, detection engine, and `/aka:*` commands. 
    
    This page covers the architecture and the Claude Code **terminal CLI** workflow (installation, hook internals, building from source). If you're running inside Claude Desktop, see the [Claude Desktop guide](claude-desktop.md) for its Settings-based install flow — everything else on this page still applies.

## Plugin structure

A Claude Code plugin is a directory that Claude Code loads at startup. AKA's layout:

```text
apps/plugin-claude-code/
├── .claude-plugin/
│   └── plugin.json      # manifest — ONLY this file goes in .claude-plugin/
├── hooks/hooks.json     # hook → script wiring (auto-discovered)
├── commands/            # /aka:* slash commands (auto-discovered)
│   ├── setup.md         #   /aka:setup     — onboarding wizard
│   ├── dashboard.md     #   /aka:dashboard — launch the web dashboard (via the aka CLI)
│   ├── health.md        #   /aka:health    — detection activity + posture
│   ├── findings.md      #   /aka:findings  — recent findings (masked)
│   ├── recommend.md     #   /aka:recommend — top recommendations
│   ├── audit.md         #   /aka:audit     — recent enforcement decisions
│   ├── scan.md          #   /aka:scan      — scan working tree for code flaws
│   ├── tokens.md        #   /aka:tokens    — token usage + estimated cost
│   └── exceptions.md    #   /aka:exceptions — active detection exceptions (read-only)
└── scripts/*.js         # built scripts (tsup output, self-contained):
                         #   hooks + query.js (read surface) + dashboard.js + onboard.js
```

The manifest is intentionally minimal. `name` becomes the command namespace, so `commands/setup.md` is invoked as `/aka:setup`:

```json
{
  "name": "aka",
  "version": "0.0.1",
  "description": "AKA AI-TC — inspect and govern AI prompts in Claude Code",
  "author": { "name": "AKA" }
}
```


!!! warning
    **Do not add `hooks` / `commands` keys pointing at the default folders.** 
    
    Claude Code auto-discovers `hooks/hooks.json` and `commands/`; manifest keys are only for **non-default** paths, and their formats are strict — `commands` must be a `./`-prefixed array (`["./custom/cmds/"]`) and `hooks` a `./`-prefixed string (`"./config/hooks.json"`). 
    
    The malformed `"hooks": "hooks/hooks.json"` / `"commands": "commands/"` fails manifest validation. Check with `claude plugin validate apps/plugin-claude-code`.

## Architecture in one paragraph

Every hook invocation is a **fresh, short-lived process**: Claude Code pipes a JSON event to the script's stdin and reads a JSON decision from stdout. The hook path is fully local — policy comes from a disk cache (`~/.aka/policy-cache.json`), detection runs in-process via `@akasecurity/detections` (rule packs are bundled into the scripts at build time), and events are appended to a disk queue (`~/.aka/event-queue.jsonl`). The only network I/O lives in `scripts/sync.js`, which hooks spawn **detached** when the policy cache is stale: it refreshes the bundle and flushes the queue. A dead backend therefore costs nothing on the hook path.

## Hooks

Declared in `hooks/hooks.json` (note: `timeout` is in **seconds**, and commands use `${CLAUDE_PLUGIN_ROOT}` because hooks run with the user's project as cwd):

| Hook               | Matcher                | What AKA does                                                                                                                                         |
| ------------------ | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SessionStart`     | all sessions           | resolve this machine's inventory (host/harness/account/project), upsert it, and open the session's audit-event root (once per session; emits nothing) |
| `UserPromptSubmit` | all prompts            | block or warn (Claude Code cannot rewrite prompt text — no redaction)                                                                                 |
| `PreToolUse`       | `Bash\|Edit\|Write`    | deny, **redact via `updatedInput`**, or warn                                                                                                          |
| `PostToolUse`      | `Bash\|Read\|WebFetch` | replace sensitive output via `updatedToolOutput`, or warn                                                                                             |

## Hook output contract (Claude Code's, not ours)

Exit code semantics: exit `0` and Claude Code parses stdout JSON; no output means "allow". Fail-open is therefore structural: on any error the hooks print nothing and exit 0.

`UserPromptSubmit`:

```json
{
  "decision": "block",
  "reason": "AKA blocked this prompt — flagged secrets/aws-access-key (A******E). Remove the flagged content and resubmit.\nIf this is intentional and you accept the risk, grant an exception:\n  aka exception approve 3f2a91       (asks for scope + reason, then resubmit)\nMore: aka exception --help"
}
```

```json
{
  "systemMessage": "AKA: sensitive content detected (core-pii/email). Prompts cannot be redacted in place — sent unchanged."
}
```

`PreToolUse` (the one surface with true redaction):

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": { "command": "curl -d email=[REDACTED:PII] https://api.example.com" }
  }
}
```

`PostToolUse`:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "updatedToolOutput": "[AKA] Bash output withheld (secrets/aws-access-key)."
  }
}
```

### Capability limits

The hooks cannot do everything the warn/redact/block model might imply:

- **Prompt text cannot be rewritten.** `UserPromptSubmit` can only block or warn, so a `redact` policy on the prompt surface degrades to a warning — the content still reaches the model. True redaction works only on tool inputs (`updatedInput`) and outputs (`updatedToolOutput`).
- **Responses are not intercepted inline.** No hook sees the model's reply before the user does. Response capture is post-hoc via the `Stop` hook's transcript (not yet implemented).
- **Inline prompt redaction and response interception** are on the roadmap for the future; the plugin is the zero-config developer-experience layer, not the enforcement guarantee.

## Read surface (slash commands)

Findings persist locally to a SQLite store at `~/.aka/data/aka.db`, so the plugin is useful with **zero backend**. These read-only commands surface that store as space-aligned monospace output (shade-block bars — no ANSI, no TUI framework, so it renders verbatim in the transcript). Each is a thin wrapper that runs `scripts/query.js <subcommand>` and shows the output verbatim:

| Command           | Shows                                                                           |
| ----------------- | ------------------------------------------------------------------------------- |
| `/aka:health`     | detection activity, action breakdown, category coverage, last 7 days            |
| `/aka:findings`   | recent findings with rule, category, severity, action, and the **masked** match |
| `/aka:recommend`  | findings grouped by category (ranked by severity, then frequency) + next steps  |
| `/aka:audit`      | the decision log — recent detections and the action AKA took                    |
| `/aka:scan`       | scan working-tree source files for insecure code patterns (OWASP Top 10)        |
| `/aka:tokens`     | token usage per provider/model (reconciled from transcripts) + estimated cost   |
| `/aka:exceptions` | active detection exceptions — masked value, rule, scope, expiry, uses           |
| `/aka:detections` | installed detection packs — version, rules, enabled, policy, update available   |

`/aka:exceptions` is deliberately **read-only**: creating, approving, or revoking an exception happens only out-of-band in a terminal via the `aka` CLI, never from inside a Claude Code session — the session being policed must not be able to grant its own bypass.

`/aka:detections` is read-only for the same reason: it shows each pack's installed version against the latest one the running plugin ships, and whether an update is pending — but **applying** an update happens out-of-band, via `aka detections update` in a terminal or the dashboard's **Update** button. Detection updates are never applied automatically: a plugin upgrade records what's newly available, and the installed packs keep scanning unchanged until you update them. The plugin scans with the **installed snapshot** (enabled packs only), so both the enable/disable toggle and pack versions genuinely control what runs.

## When a detection blocks: exception guidance

A block (or redact) message now guides the user toward the sanctioned escape hatch instead of a dead end. Removing the flagged content remains the primary recommendation; when the block is intentional (say, a temporary credential the user will rotate), the message includes a copy-paste-complete command:

```
AKA blocked this prompt — flagged secrets/aws-access-key (A******E). Remove the flagged content and resubmit.
If this is intentional and you accept the risk, grant an exception:
  aka exception approve 3f2a91       (asks for scope + reason, then resubmit)
More: aka exception --help
```

The masked preview reveals only the first and last character of the detected value (`A******E`), and the reference (`3f2a91`) points at a short-lived, fingerprint-only record of what was blocked, so `aka exception approve` can create the grant without the user ever retyping the secret. The approved value itself is never stored — only a keyed fingerprint and a masked preview. See [Adding exceptions](../getting-started/exceptions.md) for the full walkthrough.

The raw matched secret is **never** stored or shown — only a masked form. All are read-only; they never mutate the database. They also work directly:

```bash
node apps/plugin-claude-code/scripts/query.js findings
```

`/aka:scan` re-runs are cheap: every scanned file — including clean ones — is tracked in a local scan ledger (path + mtime + content hash), so an unchanged file is skipped without even being re-read. The ledger is keyed to a fingerprint of the active detection ruleset, so installing or updating a rule pack automatically rescans everything; findings themselves are deduplicated by content hash, so a re-run never duplicates them.

Files excluded by the repo's `.gitignore` are **still scanned** — local scratch and generated files are a common hiding place for real secrets — but their events carry `metadata.gitignored: true` and the scan summary reports those findings as informational, leaving enforcement policy to decide how loudly to treat them.

To exclude paths from scanning **entirely**, add a `.akaignore` file (gitignore syntax, honored at any directory level, deeper files and `!` negations override shallower ones). Unlike `.gitignore` it is a hard skip: matching files are never read, never ledgered, and produce no findings. A `.akaignore` negation also overrides the scanner's built-in default skip list (`node_modules`, `dist`, `vendor`, `target`, …) — e.g. `!vendor/` re-includes first-party code that lives under `vendor/`. The two files compose: `.gitignore` decides how loudly a finding is reported, `.akaignore` decides whether the file is looked at at all.

`/aka:scan --discover` (the multi-repo sweep) searches for git repositories under the **current directory** by default — never the home directory implicitly. A broader sweep requires an explicit `--root <path>` (e.g. `--root ~`), which the `/aka:scan` command only passes after naming the scope to the user and getting confirmation; `--depth <n>` bounds the recursion (default 4).

### `/aka:dashboard` — the web dashboard

For the full graphical view, `/aka:dashboard` launches the OSS web-ui (the same one `aka dashboard` opens) against your local store and points the browser at `http://localhost:4319/security`. It runs `scripts/dashboard.js`, which delegates to the `aka` CLI: the web server is bundled in [`@akasecurity/cli`](../getting-started/cli.md), not the plugin, so the launcher spawns `aka dashboard` in the background and returns immediately (`--port <N>` is forwarded). If the `aka` CLI isn't installed it prints how to get it instead of failing.

## Configuration

`/aka:setup` is a three-step onboarding wizard that records local preferences to `~/.aka/settings/settings.json` (created `0600`, owner-only) via `scripts/onboard.js`:

```json
{
  "specVersion": 2,
  "policy": "redact",
  "historicalAccess": "session-only",
  "onboardedAt": "2026-06-18T..."
}
```

- **`policy`** — `redact` (replace sensitive values in place where the host allows) or `warn` (surface a warning, never modify content).
- **`historicalAccess`** — `session-only` (default) or `full`. Consent for scanning pre-install surfaces — scratch/temp files, agent memory and prior conversation transcripts — for already-leaked secrets. Defaults to `session-only` so historical scanning is always an explicit opt-in, never an assumed grant on upgrade; declining still leaves AKA the working tree, the live session, git history and pointed scans to review.

The settings file is **versioned** (`specVersion`): each onboarding step is one more optional field, and an older `settings.json` still parses with the missing key taking its default — so a file written before `historicalAccess` existed loads as `session-only`.

### Historical backfill scan

When `historicalAccess: full` is chosen, `/aka:setup` runs `scripts/backfill.js` at the end of onboarding. It sweeps prior Claude Code transcripts (`~/.claude/projects/*/*.jsonl`, all projects, last 30 days), extracts user prompts and assistant replies, and feeds each through the **same** detect→mask→record path the live hooks use — so secrets that leaked _before_ AKA was installed surface in `/findings` alongside live ones. Findings carry their original transcript timestamp (not scan time), and only messages that actually match a rule are persisted. Tool inputs/outputs are not yet scanned. The scan is **idempotent** — it loads the store's existing content hashes and skips any message already recorded — so re-running `/aka:setup` re-scans without ever duplicating findings, and a cleared store re-scans in full. Fully fail-open. After install, no backfill is needed — the `UserPromptSubmit`/`PreToolUse`/`PostToolUse` hooks already see everything.

### Configuration-inventory scan (skills & hooks)

Once per session (on `SessionStart`, under the same once-per-session claim as the inventory pass), the plugin scans the machine's Claude Code **configuration surface** and records it in the local store:

- **Hooks** from `~/.claude/settings.json` (user scope), `<project>/.claude/settings.json` (project), `<project>/.claude/settings.local.json` (local), and each installed plugin's `hooks/hooks.json` (attributed to its plugin via `~/.claude/plugins/installed_plugins.json` — never by guessing directory layouts).
- **Skills** from `~/.claude/skills/` (personal), the project's `<project>/.claude/skills/` and top-level `<project>/skills/`, installed plugins' `skills/` directories, and every registered marketplace's skills (its root `skills/` plus each plugin's `plugins/*/skills/` and `external_plugins/*/skills/`, from `~/.claude/plugins/known_marketplaces.json`) — name/description/version from `SKILL.md` frontmatter, freshness from file mtime. Claude Code's built-in `claude-plugins-official` catalog (matched by name, or its canonical `anthropics/claude-plugins-official` repo even if the marketplace is renamed locally) is **excluded** — it is the tool's own bundled skills, not the user's config. A marketplace the user deliberately added, including Anthropic-published ones like `anthropics/skills`, **is** surfaced. Skills are de-duplicated by inventory identity (source + name), so a personal `pdf` and a marketplace `pdf` remain distinct rows.

Everything is **read-only**: the scan never edits config, and never captures environment values or secrets — hook commands, matchers, scopes, and skill metadata only. Each artifact becomes a content-addressed inventory row (re-scans update it in place; an uninstalled artifact simply goes stale and stops rendering), and each scan writes one `config_scan` audit event recording counts and any files that failed to parse. A malformed settings file never breaks the scan — the remaining sources are still collected, and the failure is noted on the scan event. Fully fail-open, like every hook path.

Environment variables are **not** used: hooks are processes spawned by Claude Code, so a slash command cannot inject env into them — a file is the only reliable channel.

Unconfigured is a valid state: until `/aka:setup` runs, detection still uses the bundled rule packs and default actions (`secret` → block, `pii` → redact where possible), and the first prompt nudges the user to run the wizard.

## Reading your local data

The plugin writes `~/.aka/data/aka.db` and the OSS web dashboard
reads that same store directly (`@akasecurity/persistence`, Server Components) — web
dashboards over your local data with nothing leaving the machine and no server to
run. Launch it with `aka dashboard` (see the [CLI](../getting-started/cli.md)).

## Installing, developing, and testing

Loading the plugin needs the Claude Code **CLI** (the `claude` command). Install it with the native installer (a standalone binary, independent of your Node setup) or npm:

```bash
curl -fsSL https://claude.ai/install.sh | bash   # recommended — native binary
# or: npm i -g @anthropic-ai/claude-code
```

Hooks invoke `node` at runtime, so `node` must be on the user's PATH — note this as an install prerequisite.

Load the plugin for a single session without a marketplace install:

```bash
pnpm turbo run build --filter=@akasecurity/plugin-claude-code   # tsup bundles src → scripts/*.js (self-contained)
claude --plugin-dir ./apps/plugin-claude-code           # load for one session
# after a rebuild, run /reload-plugins inside the session
```

Inside the session, `/hooks` lists the registered hooks (confirm `UserPromptSubmit`, `PreToolUse`, `PostToolUse`) and `/aka:setup` runs the onboarding wizard. Submitting a prompt containing a fake AWS key (`AKIAIOSFODNN7EXAMPLE`) should be blocked with the AKA reason — then `/aka:findings` and `/aka:health` show the recorded detection.

## Distribution (marketplace + npm)

End users do **not** clone this repo. The plugin is distributed as a published npm package, listed in a Claude Code marketplace catalog (`.claude-plugin/marketplace.json` at the repo root):

```bash
/plugin marketplace add akasecurity/marketplace
/plugin install ai-tc@akasecurity
```

Run these in the Claude Code **terminal CLI** (`claude`) — `/plugin` marketplace management is not exposed in the IDE/editor extension surfaces. Running Claude Desktop instead? See [Claude Desktop → Installing](claude-desktop.md#installing) for the equivalent Settings-panel flow.

Removing the plugin later? See [Uninstalling](uninstalling.md).

If you have the [`aka` CLI](../getting-started/cli.md) installed, `aka plugins install claude-code` runs both steps for you (it delegates to the same `claude plugin` manager), and `aka update claude-code` updates the plugin later. Either way, restart Claude Code afterwards to load the change.

> **Pre-release distribution — private GitHub Packages.** While AKA is pre-release we
> publish **only private artifacts to GitHub Packages** — nothing goes to the public npm
> registry. The `/plugin install` above resolves an npm source, so it can only be fetched
> with a valid `~/.npmrc` that points the `@akasecurity` scope at GitHub Packages
> **and** carries a GitHub token with `read:packages` access to our packages:
>
> ```ini
> @akasecurity:registry=https://npm.pkg.github.com
> //npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
> ```
>
> Write it once (`chmod 600 ~/.npmrc`) before running `/plugin install`; without it the
> marketplace fetch fails with `401`/`403`. This requirement disappears once the project
> is public.

Why npm rather than a git source: a Claude Code marketplace install **copies the source as-is and never runs a build step**, and the hook scripts (`scripts/*.js`) are git-ignored. A `github` / `git-subdir` source would therefore ship a plugin whose `hooks.json` points at files that don't exist. The npm source instead ships the built, self-contained bundle (the `@akasecurity/*` workspace deps and `zod` are inlined by `tsup`, so the published package has **no runtime dependencies**). `node` must still be on the user's PATH — the hooks invoke it.

Releasing is automated by [`.github/workflows/release-plugin.yml`](https://github.com/akasecurity/ai-tc/blob/main/.github/workflows/release-plugin.yml):

1. Bump `version` in both `apps/plugin-claude-code/package.json` and `.claude-plugin/plugin.json`.
2. Push a tag `plugin-v<version>` (e.g. `plugin-v0.1.0`).
3. CI builds with Turbo, verifies the tag matches both manifests, smoke-tests the fail-open contract, publishes to **private GitHub Packages** (`npm.pkg.github.com`, authenticated with the workflow's built-in `GITHUB_TOKEN`), and attaches the tarball to a GitHub Release.

The marketplace's npm source is unpinned, so `/plugin install` and `/plugin update` resolve the latest published version — the npm package version is the single source of truth.

Hooks are also testable without Claude Code:

```bash
echo '{"prompt":"key AKIAIOSFODNN7EXAMPLE"}' | node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → {"decision":"block","reason":"AKA blocked this prompt — flagged secrets/aws-access-key (A******E). Remove the flagged content and resubmit.\n..."}

echo 'not json' | node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → no output, exit 0 (fail-open)
```

## Fail-open guarantee

Three layers, all of which must hold:

1. Hook entries wrap `main()` in try/catch; the catch prints nothing and exits 0 (= allow).
2. `createPluginRuntime().processText` catches internally and returns `{ action: 'log' }`.
3. Exit-code semantics: even an unhandled crash with a non-2 exit code is non-blocking for Claude Code.

## Building a new adapter

To support a different AI tool (Cursor, Continue, etc.), implement the `AkaPluginAdapter` interface from `packages/plugin-sdk` and reuse the same runtime: `createPluginRuntime` (local scan + policy), `buildIngestEvent`, `registerRulePack`, and `syncWithBackend` (out-of-band network). See `packages/plugin-sdk/src/types.ts` for the full interface.
