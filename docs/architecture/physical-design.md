# Physical Design

## Repository layout

The open-source surface consists of the plugin, the CLI, the local SQLite store, and
the OSS web-ui. AKA also has a separate enterprise control plane for teams.

```
ai-tc/
├── apps/
│   ├── plugin-claude-code/  Hooks for Claude Code and Claude Desktop
│   ├── cli/                 The `aka` CLI — scan, stats, dashboard, plugin management
│   └── web-ui/              OSS Next.js dashboard — reads the local store directly
├── packages/
│   ├── schema/           Zod contracts — single source of truth
│   ├── config/           Env loader — sole reader of process.env
│   ├── detections/       Pure detection engine (no I/O) — runs identically
│   │                     everywhere detection happens (plugin, backend)
│   ├── scanner/          Working-tree file walking/discovery behind `/aka:scan`
│   ├── persistence/      Local SQLite store adapter — no backend required
│   ├── plugin-runtime/   Resolves the DataGateway and drives the hook
│   │                     lifecycle
│   ├── plugin-sdk/       AkaPluginAdapter interface shared by every harness
│   ├── client/           Generated API client (sole caller of fetch)
│   ├── local-ops/        Local-store maintenance — rule-pack update cache, fs scan/render
│   ├── extract/          Export helpers (CSV, …)
│   ├── telemetry/        Anonymous usage telemetry SDK
│   ├── dashboard-ui/     Shared React views — same components render the OSS
│   │                     and enterprise dashboards
│   ├── eslint-config/    Shared flat ESLint config (strict, boundary rules)
│   └── ui-kit/           Shared low-level UI primitives (buttons, cards, tables, …)
├── rules/
│   ├── core-pii/           PII detections (email, SSN, …) with fixtures
│   ├── core-financial/     Card numbers, bank routing, … with fixtures
│   ├── core-phi/           Health-record identifiers with fixtures
│   ├── core-code-context/  Code-context signals with fixtures
│   ├── secrets/            Cloud/API credentials with fixtures
│   ├── secrets-infra/      Infra-specific credentials with fixtures
│   └── code-flaws/         OWASP-style insecure code patterns with fixtures
├── tools/
│   ├── installer/        Cross-platform bootstrap script for the `aka` CLI
│   ├── migrator/         CLI for applying DB migrations
│   └── rules-publisher/  CLI that publishes rules/ packs to the registry
└── skills/
    ├── backend-conventions/SKILL.md
    └── write-detection-rule/SKILL.md
```

## Build & distribution

`apps/plugin-claude-code` and `apps/cli` are built with `tsup`, which inlines every
`@akasecurity/*` workspace dependency (`noExternal: [/^@akasecurity\//]`) — the published
package ships with **no runtime `node_modules`** of its own. Only Node itself needs to
be on the user's `PATH`, since the hooks invoke it directly. The CLI additionally
bundles the OSS web-ui as a sibling asset (copied in by `prepack`, spawned as its own
Next.js process at runtime) rather than compiling it into the CLI's own JS.

Both are versioned and released together — see
[Claude Code → Distribution](../plugins/claude-code.md#distribution-marketplace-npm)
for the exact publish pipeline (npm/GitHub Packages, marketplace manifest,
tag-triggered CI). Self-hosted enterprise ships as a single Docker container; the OSS
path covered by this doc needs no server at all.

## Runtime processes

| Process                        | Lifetime                                                                | Network                                                                  |
| ------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| Hook scripts (`scripts/*.js`)   | One per event — spawned by Claude Code/Desktop, exits after printing a decision | None — reads/writes local disk only                                        |
| `sync.js`                       | Spawned **detached**, only when the policy cache is stale                | The only hook-path process that touches the network — fetches the rule bundle, flushes queued events |
| `aka dashboard`                 | Runs until stopped — a local Next.js server (default `localhost:4319`)   | Local only; reads `~/.aka/data/aka.db` directly, no outbound calls          |
| `aka tui`                       | Runs until you quit — an Ink terminal app                                | None — opens no port at all                                                |

The hook path itself never blocks on network I/O — see
[Architecture in one paragraph](../plugins/claude-code.md#architecture-in-one-paragraph)
for why a dead backend costs nothing.

## Network boundary

Nothing leaves the machine on the hot hook path: `UserPromptSubmit`/`PreToolUse`/
`PostToolUse` only ever read a disk-cached policy bundle
(`~/.aka/policy-cache.json`) and append to a disk queue
(`~/.aka/event-queue.jsonl`) — never a live network call. Only one package in the
codebase is allowed to call `fetch` at all — `@akasecurity/client` — and that's
enforced, not just documented: an ESLint rule bans `fetch()` everywhere else, the
same way `n/no-process-env` confines `process.env` reads to `@akasecurity/config`.

## Storage

The local store is a single SQLite file (`~/.aka/data/aka.db`) via Node's built-in
`node:sqlite` — no native module to compile, no database server, single-writer.

