# System Overview

## Repository layout

```
ai-control-plane/
├── apps/
│   ├── backend/          Fastify 5 API server (per-customer control plane)
│   ├── registry/         Fastify 5 global rule marketplace (cross-tenant, public read)
│   ├── dashboard/        React 19 + Vite SPA
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
    ├── docker-compose.yml        (prod / self-hosted)
    ├── docker-compose.dev.yml    (dev)
    └── docker-compose.local.yml  (plugin + local backend over ~/.aka/data)
```

## Package dependency rules

AKA enforces strict import boundaries via the pnpm workspace graph + `tsc` (a forbidden import does not resolve / fails typecheck) and a dedicated CI gate (`pnpm check:boundaries`), not a lint plugin. Violating them is a CI failure.

```
apps/backend/routes       →  @alsoknownassecurity/schema (Zod only), services/
apps/backend/services     →  repositories/
apps/backend/repositories →  @alsoknownassecurity/schema/drizzle/*, drizzle-orm
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
3. **No Drizzle imports outside `apps/backend/src/repositories/` and `tools/migrator/`**

## Local-first plugin & optional backend (Phase 1)

The plugin is fully useful with **zero backend and zero Docker**. The runtime
depends on a single **`DataGateway`** port (defined in `@alsoknownassecurity/plugin-sdk`);
`@alsoknownassecurity/plugin-runtime` resolves the implementation from the run mode:

- **standalone** (default) → a SQLite store at `~/.aka/data/aka.db` via
  `@alsoknownassecurity/persistence` — the always-present writer of `events` and `findings`.
- **attached** → the (local) backend's HTTP API via `@alsoknownassecurity/client`; the plugin
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
   (optional) local backend (Docker)        (Phase 2) enterprise sync
   dashboards + policy engine, SAME DB        tenant-less wire → Org API
```

A user may **opt in** to "Plugin + local backend" (`docker/docker-compose.local.yml`):
the backend and dashboard read/serve that **same** `aka.db`. Two seams make the
shared file safe:

- **Migration ownership.** The plugin owns the schema and self-migrates on open,
  so the backend runs `MIGRATE_ON_START=false` — no dual-migrator race. The
  writers are disjoint (plugin → `events`/`findings`, backend → `policies`) and
  WAL lets both share the file.
- **Identity.** The plugin seeds one tenant/user with a **generated UUID** on
  first run. In `local` mode the backend resolves its tenant from that existing
  row (not a hardcoded stub), so its dashboards aren't empty over a
  plugin-populated DB. Phase-2 org-connect rebinds the identity to org-issued ids.

Enterprise auth and remote sync are **Phase 2**; in Phase 1 nothing leaves the
machine.

## Data flow

### Event ingestion (plugin → local store; → backend is Phase 2)

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
│        → queueEvent() to backend            │
│     → writes {action, prompt?} to stdout    │
│  3. Claude receives (possibly redacted)     │
│     prompt and responds                     │
└────────────────────────────────────────────┘
                    │
                    ▼ HTTP POST /v1/events
┌────────────────────────────────────────────┐
│  @alsoknownassecurity/backend                               │
│                                             │
│  Route: validate IngestRequest (Zod)        │
│  Service: ingestEvents()                    │
│  Repository: insertEvents()                 │
│    → isoToEpochMillis(ev.occurredAt)        │
│    → deduplication by event.id              │
│    → INSERT INTO events                     │
│  Response: { accepted, duplicates }         │
└────────────────────────────────────────────┘
```

### Policy bundle (backend → plugin cache)

The plugin caches a `PolicyBundle` from the backend. On each session start (or after TTL expiry), it refreshes:

```
GET /v1/policy-bundle
→ { policies, customKeywords, bundleVersion }
```

The plugin stores this in memory and uses it to resolve actions without a round-trip per event.

### Dashboard

The dashboard is a React SPA that calls the backend's REST API via TanStack Query. It builds to `apps/backend/public/` so the production container serves both the API and the UI from a single port.

## API client generation

`@alsoknownassecurity/client` is **generated**, not hand-maintained. The contract flows one way:

```
@alsoknownassecurity/schema (Zod)  →  Fastify OpenAPI spec  →  @alsoknownassecurity/client (Hey API)
  single source        GET /openapi.json         typed SDK + TanStack Query
   of truth          (backend + registry)        options, sole fetch caller
```

`packages/client/gen/dump-specs.ts` boots each Fastify app to `ready()` against an
in-memory SQLite DB and writes `app.swagger()` to `packages/client/openapi/*.json`
(no server, no real DB). [Hey API](https://heyapi.dev) (`@hey-api/openapi-ts`) then
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

## Request lifecycle (backend)

Every API request passes through:

```
Fastify onRequest hook
  → Auth stub: validates Bearer token against AKA_LOCAL_TOKEN
  → (future: Better Auth middleware with JWT + tenant resolution)
      ↓
Route handler
  → Zod validation (safeParse on body / query)
  → calls Service function
      ↓
Service
  → orchestrates one or more Repository calls
  → contains business logic
      ↓
Repository
  → Drizzle query builder (select / insert / update)
  → no business logic
      ↓
Response
  → plain object → JSON serialization by Fastify
```

## Database schema

### Catalog / Tenant split

The Drizzle schema lives in `packages/schema/src/drizzle/` and is split into two logical groups:

| Group       | Tables                                                                                                                                              | File pair                                  |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| **Catalog** | `tenants`, `users`                                                                                                                                  | `catalog/sqlite.ts`, `catalog/postgres.ts` |
| **Tenant**  | `events`, `findings`, `policies`, `inventory`, `source_project`, `audit_events`, `classified_data`, `inspection_definitions`, `inspection_findings` | `tenant/sqlite.ts`, `tenant/postgres.ts`   |

The catalog tables are cross-tenant metadata (no row-level security on either dialect). The tenant tables hold per-tenant operational data and are protected by RLS on Postgres.

The combined runtime objects `sqliteSchema` / `pgSchema` (exported from `@alsoknownassecurity/schema/drizzle`) spread both groups together and are used as the drizzle client's `{ schema }` argument.

| Table      | Purpose                                                     |
| ---------- | ----------------------------------------------------------- |
| `tenants`  | Top-level billing/auth boundary                             |
| `users`    | Member of a tenant, with a role                             |
| `events`   | Each captured prompt / response / code_change               |
| `findings` | Each rule match on an event                                 |
| `policies` | Per-tenant action overrides (allow / warn / redact / block) |

#### Generalized data model (additive)

A second set of tenant tables generalizes `events`/`findings`/rules into a
star-shaped data model. They are **additive**: provisioned by the same
migrations and populated by dedicated repositories in `@alsoknownassecurity/persistence` (and the
facade's `ensureInventory`), but the live capture path still writes
`events`/`findings` — the writer cutover happens in a later phase.

| Table                    | Role                                                                                 |
| ------------------------ | ------------------------------------------------------------------------------------ |
| `inventory`              | Existence/dimension rows (`host`, `harness`, `user`), content-addressed and deduped  |
| `source_project`         | The repository/project a session ran against, content-addressed by remote url        |
| `audit_events`           | Timeline/fact rows forming a self-referential tree (`session → run → tool_call → …`) |
| `classified_data`        | Small class dimension of recognized sensitive-data kinds (`aws_key`, `email_pii`, …) |
| `inspection_definitions` | A detection rule version (id encodes the version, so a finding cites the exact rule) |
| `inspection_findings`    | A hit of a definition against an audit event                                         |

Inventory/source rows carry content-addressed, tenant-scoped ids
(`sha256(tenant_id + …)`, computed in `@alsoknownassecurity/persistence`) so repeat sessions
upsert idempotently. Hot filter keys (`os_version`, `harness_version`) are SQLite
generated columns over the JSON `attributes` bag, indexed for facets.

### Timestamp representation and the `time.ts` boundary

SQLite cannot store `TIMESTAMP WITH TIME ZONE` natively. The storage strategy is:

| Dialect            | Column type                | Wire format     |
| ------------------ | -------------------------- | --------------- |
| SQLite             | `integer` (epoch-millis)   | —               |
| Postgres           | `timestamp with time zone` | —               |
| Zod (API boundary) | `z.string().datetime()`    | ISO-8601 string |

**Who converts and where:** `packages/schema/src/time.ts` exports two pure helpers with no Drizzle import:

- `isoToEpochMillis(iso: string): number` — used by the repository WRITE path before inserting `occurred_at` into SQLite
- `epochMillisToIso(ms: number): string` — used by the repository READ path when mapping rows back to the Zod `Event` type

The boundary is strictly at the repository layer (`apps/backend/src/repositories/events.ts`). All routes and services above the repository see ISO-8601 strings. The raw integer never escapes past `listEvents`.

```
Zod(ISO) → repo.isoToEpochMillis → sqlite INTEGER
           repo.epochMillisToIso ← sqlite INTEGER
Zod(ISO) ← repo
```

### Row-level security (Postgres only)

Every tenant table (`events`, `findings`, `policies`, and the generalized data-model tables `inventory`, `source_project`, `audit_events`, `classified_data`, `inspection_definitions`, `inspection_findings`) declares `ENABLE ROW LEVEL SECURITY` and a single permissive policy keyed on `current_setting('app.tenant_id', true)` in the Postgres dialect. The SQLite tables have no RLS declarations (single-tenant local mode).

The generated migration (`drizzle/postgres/0000_*.sql`) contains:

- `ALTER TABLE "events" ENABLE ROW LEVEL SECURITY;`
- `CREATE POLICY "events_tenant_isolation" ... USING ("tenant_id" = current_setting('app.tenant_id', true))`
- (same for `findings` and `policies`)

**FORCE ROW LEVEL SECURITY companion:** drizzle-kit 0.31 emits `ENABLE ROW LEVEL SECURITY` and `CREATE POLICY` but NOT `FORCE ROW LEVEL SECURITY`. Without FORCE, the table owner role bypasses RLS entirely. FORCE is added by **journaled custom migrations** (e.g. `drizzle/postgres/0001_force_rls.sql`, and `0010_force_rls_meta_data_model.sql` for the generalized data-model tables), recorded in `meta/_journal.json` and applied by the standard drizzle-orm migrator after the table-creating migration they accompany. These are the only hand-written SQL files in the project. (Note: a plain file sorted by name — e.g. a `9999_*.sql` — would NOT be applied, because the migrator reads the journal, not the directory listing.) When drizzle-kit gains native FORCE RLS support, this migration should be removed and `0000_initial` regenerated.

**Per-request tenant context.** `withTenantContext` opens a transaction and issues `set_config('app.tenant_id', $1, true)` (tenantId bound, never interpolated) so Postgres FORCE RLS resolves the active tenant; SQLite is a passthrough. Routes never call it directly. A **tenant-scope plugin** (`apps/backend/src/plugins/tenant-scope.ts`) decorates each request with `req.tenantScope` — an `open` `TenantScope` bound to that request's tenant — and repositories open their own context from it via `runInScope`. Isolation therefore lives in the data layer: a route cannot reach the database without RLS in force, because the only db-bearing value it can touch is the scope (typed `TenantScope | null`, narrowed by `requireTenantScope` which throws loudly rather than allow an un-scoped query). For work that must be atomic across several repository calls, `withTenantTransaction` opens one transaction and yields a `joined` scope shared by every call, so they commit or roll back together.

## Multi-tenancy (stub)

The current auth hook is a single-user bearer token stub. Real multi-tenancy (Better Auth + row-level tenantId filtering via `SET LOCAL app.tenant_id`) is planned for Week 3. The schema and repository layer are already tenancy-aware — every row carries a `tenant_id`, RLS policies are declared in the Postgres dialect, and the FORCE companion migration is in place.

## Observability (OpenTelemetry)

The system is instrumented with OpenTelemetry for distributed tracing, metrics, and trace-correlated logs, following CNCF conventions. It is **off by default and zero-cost when disabled** (`OTEL_ENABLED=false`): the SDK is loaded via a `node --import` preload that returns before importing any `@opentelemetry/*` code unless enabled, and hot-path code (the DB span wrapper, outgoing header injection) short-circuits on a boolean.

- **Collector-centric.** Apps only ever emit OTLP to an OpenTelemetry Collector, which fans out to Jaeger (traces) + Prometheus (metrics) by default, or to AWS CloudWatch/X-Ray or Azure Monitor by collector config alone — vendor choice never touches app code.
- **Coverage.** HTTP/Fastify/Postgres via auto-instrumentation; an explicit `db.query` span + metric at the `runInScope` chokepoint covers SQLite (no auto-instrumentation) and Postgres; auth lookups get spans. `@alsoknownassecurity/client` injects W3C `traceparent` + a correlation id on every outgoing call, so plugin→backend→DB→registry is one trace.
- **Correlation.** Every request reuses/echoes `x-request-id` (the trace id when present) and every log line carries `trace_id`/`span_id`.
- **Probes.** `/healthz/live` is a dependency-free liveness check; `/healthz/ready` is dependency-aware readiness (`503` when the DB ping fails). Both are always available regardless of `OTEL_ENABLED`.

Implementation lives in `@alsoknownassecurity/telemetry` (shared SDK bootstrap) with per-service `--import` preloads; the runtime keeps `@opentelemetry/*` external because its instrumentation uses dynamic `require()`s a bundle cannot express. See [Observability](../operations/observability.md).

## Follow-ups (known gaps)

**Dialect-explicit schema type aliases:** the former `CatalogSchema`/`TenantSchema` aliases were SQLite-typed only, so a Postgres handle had no type to describe it. They are replaced by dialect-explicit aliases in `@alsoknownassecurity/schema/drizzle`: `SqliteSchema`/`PgSchema` (the full combined handles) and the logical seams `SqliteCatalogSchema`, `SqliteTenantSchema`, `PgCatalogSchema`, `PgTenantSchema`. Each is derived with `Pick<...>` over the runtime `sqliteSchema`/`pgSchema` objects, so the types cannot drift from what `drizzle()` actually receives — a misspelled or removed table is a compile error, and new tables are tracked automatically. The dialect is part of the name because a Postgres table is not assignable to a SQLite one. Locked by type-level assertions in `packages/schema/src/drizzle/schema-types.test.ts` (enforced by `tsc --noEmit`).

**Server-derived `findings.tenantId`:** `tenantId` is **server-derived** — injected from the authenticated context, never accepted from a request body. `packages/schema/src/zod/finding.ts` now mirrors the events pattern: the full `Finding` row schema carries `tenantId` (restoring Zod↔Drizzle parity with `findings.tenant_id`), and `DetectedFinding = Finding.omit({ tenantId: true })` is the producer-side shape. The future findings repository injects `tenantId` from the tenant scope on write — exactly as `insertEvents(scope, batch, userId)` reads `scope.tenantId` for `IngestEvent` — giving a single source of truth that lines up with the Postgres RLS `WITH CHECK (tenant_id = current_setting('app.tenant_id'))`. Decision locked by `packages/schema/src/zod/finding.test.ts`.
