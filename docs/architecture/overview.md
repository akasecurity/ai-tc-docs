# System Overview

## Repository layout

```
ai-control-plane/
├── apps/
│   ├── backend/          Fastify 5 API server (enterprise, per-customer control plane)
│   ├── registry/         Fastify 5 global rule marketplace (cross-tenant, public read)
│   ├── dashboard/        React 19 + Vite SPA (enterprise)
│   ├── docs/             This MkDocs site
│   └── plugin-claude-code/  Claude Code hook scripts
├── packages/
│   ├── schema/           Zod contracts — single source of truth
│   ├── config/           Env loader — sole reader of process.env
│   ├── detections/       Pure detection engine (no I/O)
│   ├── client/           Generated API client (sole caller of fetch)
│   ├── plugin-sdk/       AkaPluginAdapter interface + runtime
│   └── ui-kit/           Shared React component library (stub)
├── rules/
│   ├── core-pii/         Email + SSN rules with fixtures
│   └── secrets/          AWS key + GitHub PAT rules with fixtures
├── tools/
│   ├── migrator/         CLI for applying DB migrations
│   └── rules-publisher/  CLI that publishes rules/ packs to the registry
├── skills/
│   ├── backend-conventions/SKILL.md
│   └── write-detection-rule/SKILL.md
└── docker/
    ├── Dockerfile
    ├── docker-compose.yml        (prod / self-hosted — app + Postgres)
    └── docker-compose.dev.yml    (dev — Postgres + backend + dashboard + docs)
```

`apps/backend` and `apps/dashboard` are the enterprise control plane — see the
enterprise docs for their architecture. Everything below describes the
open-source surface: the plugin, the CLI, the local SQLite store, and the OSS
web-ui.

## Package dependency rules

AKA enforces strict import boundaries via the pnpm workspace graph + `tsc` (a forbidden import does not resolve / fails typecheck) and a dedicated CI gate (`pnpm check:boundaries`), not a lint plugin. Violating them is a CI failure.

```
apps/plugin-*             →  @alsoknownassecurity/plugin-sdk, @alsoknownassecurity/plugin-runtime
@alsoknownassecurity/plugin-runtime       →  @alsoknownassecurity/plugin-sdk, @alsoknownassecurity/persistence, @alsoknownassecurity/client, @alsoknownassecurity/schema
@alsoknownassecurity/plugin-sdk           →  @alsoknownassecurity/detections, @alsoknownassecurity/client, @alsoknownassecurity/persistence, @alsoknownassecurity/schema
@alsoknownassecurity/persistence          →  node:sqlite, @alsoknownassecurity/schema (no Drizzle, no @alsoknownassecurity/detections)
@alsoknownassecurity/client               →  fetch (only package allowed to call fetch)
@alsoknownassecurity/config               →  process.env (only package allowed to read env)
@alsoknownassecurity/detections           →  @alsoknownassecurity/schema (no I/O, no Node-API deps)
```

Three cross-cutting rules every contributor must remember:

1. **No `process.env` outside `packages/config`** — enforced by `n/no-process-env: 'error'`
2. **No `fetch()` outside `packages/client`** — by convention + boundary lint
3. **No Drizzle imports outside `apps/backend/src/repositories/` and `tools/migrator/`** (the enterprise repository layer — see the enterprise docs)

## Local-first plugin & optional backend (Phase 1)

The plugin is fully useful with **zero backend and zero Docker**. The runtime
depends on a single **`DataGateway`** port (defined in `@alsoknownassecurity/plugin-sdk`);
`@alsoknownassecurity/plugin-runtime` resolves the implementation from the run mode:

- **standalone** (default) → a SQLite store at `~/.aka/data/aka.db` via
  `@alsoknownassecurity/persistence` — the always-present writer of `events` and `findings`.
- **attached** → the enterprise backend's HTTP API via `@alsoknownassecurity/client`; the plugin
  also pulls the org's centrally-managed ruleset into a cache and detects with it.

Detection runs in-process in either mode, and results surface through slash
commands (`/aka:health`, `/aka:findings`, `/aka:recommend`, `/aka:audit`). The
gateway and the shared data shapes live in `@alsoknownassecurity/plugin-sdk` / `@alsoknownassecurity/persistence`,
so a new plugin (Claude Code first, VSCode/Cursor later) adds only a thin
tool-specific adapter, never a copy of the storage/detection logic.

```
 Claude Code ──hooks──▶ AKA plugin (adapter) ──▶ @alsoknownassecurity/plugin-sdk
                                                      │ writes
                                                      ▼
        ~/.aka/data/aka.db   (events · findings · policies)   ◀── shared by all plugins
                ▲                                   ▲
   OSS web-ui reads it directly            (Phase 2) enterprise sync
   (@alsoknownassecurity/persistence, no server)       tenant-less wire → Postgres backend (HTTP)
```

For web dashboards over local data with no server, the **OSS web-ui** reads that
same `aka.db` directly (`@alsoknownassecurity/persistence`, Server Components) — there is no
local _enterprise backend_. The enterprise backend is Postgres-only; the plugin
syncs to it over HTTP in `attached` mode (Phase 2), never over a shared SQLite
file. In Phase 1 nothing leaves the machine.

## Data flow

### Event ingestion (plugin → local store; enterprise sync is a separate, later hop)

```
┌────────────────────────────────────────────┐
│  Claude Code session                        │
│                                             │
│  1. User types prompt                       │
│  2. UserPromptSubmit hook fires             │
│     → user-prompt-submit.js reads stdin     │
│     → createPluginRuntime().processText()   │
│        → scan() against loaded rules        │
│        → resolve policy action              │
│        → redact if action = redact          │
│        → writeEvent() to the local store    │
│     → writes {action, prompt?} to stdout    │
│  3. Claude receives (possibly redacted)     │
│     prompt and responds                     │
└────────────────────────────────────────────┘
                    │
                    ▼
        ~/.aka/data/aka.db  (@alsoknownassecurity/persistence)
```

In `attached` mode the same event is additionally queued to the enterprise
backend over HTTP — see the enterprise docs for that request's lifecycle
(route → service → repository) on the backend side.

### Policy bundle (attached mode only)

In `attached` mode the plugin caches a `PolicyBundle` pulled from the
enterprise backend and refreshes it on session start (or after TTL expiry),
so it can resolve actions without a round-trip per event. In `standalone`
mode there is no bundle to fetch — policy comes from the local store.

### Dashboard

The OSS **web-ui** (`apps/web-ui`, Next.js) reads the local SQLite store
directly through `@alsoknownassecurity/persistence` in Server Components — no
`@alsoknownassecurity/client`, no HTTP, no rule registry. It renders the same
presentational `*View` components from `@alsoknownassecurity/dashboard-ui`
that the enterprise dashboard does; enterprise-only affordances are injected
as optional props (e.g. `FindingDetailView`'s `footer`,
`DetectionDetailView`'s action callbacks) rather than forking the view. The
**Detections** page adds a matching OSS read path (`db().detections` —
list/detail/stats over `installed_packs`, reusing the pure
`buildDetectionsList` / `rowToDetectionDetail` builders from
`@alsoknownassecurity/schema` so shapes never drift from the hosted contract)
plus the web-ui's **first local write path**: changing a detection's
enforcement policy (or toggling it) is a Next.js Server Action that calls the
persistence write facade and `revalidatePath`s the page. Import-from-library
and pulling upstream updates remain enterprise-only (they require the
registry).

The **Policies** page renders the shared built-in policy catalog (`db().policyCatalog` — monitor/warn/redact/block with live "used by N detections" counts joined from `installed_packs`, NULL policy attributed to Monitor to match the Detections views) and, below it, the read-only local enforcement config (`db().policies.readPolicies()`) so a disabled per-detection-type policy — which the plugin still resolves against — is never invisible. The **Data Shares** page reads the local egress register through `db().shares` (grouped destinations, the needs-review strip, and the selected destination detail), with the egress Block/Allow decision persisted via a Server Action (`setEgressDecision`, which returns whether the row still existed so the client can surface a failed write). The **Inventory** page reads the asset model through `db().inventoryAssets` (harnesses / assets / projects / stats + the selected node's project file tree / harness overview / asset detail / file drawer), resolving the active selection with the shared `resolveInventorySelection` so it can't fork from enterprise; the per-file LLM-access and MCP-trust edits are Server Actions (`setFileAccess` / `setMcpTrust`) that likewise return success so the client surfaces failures. Both Data Shares and Inventory have no scanner yet, so they ship a **removable sample dataset**: `db().seedSampleData()` runs once per process (guarded by a persistent `app_meta` marker + domain-emptiness, so it never seeds over real data), and a `hasSampleData()`-gated **Clear sample data** action calls `clearSampleData()` (deletes every `provenance='sample'` row, children first) and `revalidatePath('/', 'layout')` since the clear crosses domains.

## API client generation

`@alsoknownassecurity/client` is **generated**, not hand-maintained. The contract flows one way:

```
@alsoknownassecurity/schema (Zod)  →  Fastify OpenAPI spec  →  @alsoknownassecurity/client (Hey API)
  single source        GET /openapi.json         typed SDK + TanStack Query
   of truth          (backend + registry)        options, sole fetch caller
```

`packages/client/gen/dump-specs.ts` boots each Fastify app to `ready()` against a
never-connected Postgres config and writes `app.swagger()` to
`packages/client/openapi/*.json` (route registration only — no server, no real DB). [Hey API](https://heyapi.dev) (`@hey-api/openapi-ts`) then
generates `packages/client/src/generated/**` — **types only**, no runtime
validation, since the server already validates and the types derive from the same
Zod contracts. A thin hand-written wrapper (`src/index.ts`) keeps the stable public
surface (`createClient` / `createRegistryClient` / `RegistryRequestError`, typed
with `@alsoknownassecurity/schema`) and owns what codegen can't: the dual `x-api-key` + `Bearer`
auth headers and per-instance `baseUrl`/`token`.

Specs and generated output are committed; `pnpm --filter @alsoknownassecurity/client gen:openapi:check`
regenerates and fails on drift (run in CI). Generated files are `@ts-nocheck`,
ESLint-ignored, and Prettier-ignored — type-safety is enforced at the wrapper
boundary and each call site.

## Enterprise backend internals

The enterprise backend's request lifecycle (Fastify route → service →
Drizzle repository), its Postgres schema (catalog/tenant table split, the
generalized inventory/audit data model), the `time.ts` SQLite↔Postgres
timestamp boundary, Row-Level Security enforcement, multi-tenancy, and
OpenTelemetry instrumentation are covered in the enterprise docs — none of it
applies to the OSS local-first path described above.
