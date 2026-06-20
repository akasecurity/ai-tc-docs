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
│   ├── client/           Typed fetch client (sole caller of fetch)
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

AKA enforces strict import boundaries via `eslint-plugin-boundaries`. Violating them is a CI failure.

```
apps/backend/routes       →  @aka/schema (Zod only), services/
apps/backend/services     →  repositories/
apps/backend/repositories →  @aka/schema/drizzle/*, drizzle-orm
apps/plugin-*             →  @aka/plugin-sdk
@aka/plugin-sdk           →  @aka/detections, @aka/client, @aka/schema
@aka/client               →  fetch (only package allowed to call fetch)
@aka/config               →  process.env (only package allowed to read env)
@aka/detections           →  @aka/schema (no I/O, no Node-API deps)
```

Three cross-cutting rules every contributor must remember:

1. **No `process.env` outside `packages/config`** — enforced by `n/no-process-env: 'error'`
2. **No `fetch()` outside `packages/client`** — by convention + boundary lint
3. **No Drizzle imports outside `apps/backend/src/repositories/` and `tools/migrator/`**

## Local-first plugin & optional backend (Phase 1)

The plugin is fully useful with **zero backend and zero Docker**. It owns a local
SQLite store at `~/.aka/data/aka.db` and is the always-present writer of `events`
and `findings`; detection runs in-process and results surface through slash
commands (`/aka:health`, `/aka:findings`, `/aka:recommend`, `/aka:audit`). This
on-disk layout and the data layer live in `@aka/plugin-sdk` and are shared by
every plugin (Claude Code first, VSCode/Cursor later) — a new plugin adds only a
thin tool-specific adapter, never a copy of the storage/detection logic.

```
 Claude Code ──hooks──▶ AKA plugin (adapter) ──▶ @aka/plugin-sdk
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
│  @aka/backend                               │
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

| Group       | Tables                           | File pair                                  |
| ----------- | -------------------------------- | ------------------------------------------ |
| **Catalog** | `tenants`, `users`               | `catalog/sqlite.ts`, `catalog/postgres.ts` |
| **Tenant**  | `events`, `findings`, `policies` | `tenant/sqlite.ts`, `tenant/postgres.ts`   |

The catalog tables are cross-tenant metadata (no row-level security on either dialect). The tenant tables hold per-tenant operational data and are protected by RLS on Postgres.

The combined runtime objects `sqliteSchema` / `pgSchema` (exported from `@aka/schema/drizzle`) spread both groups together and are used as the drizzle client's `{ schema }` argument.

| Table      | Purpose                                                     |
| ---------- | ----------------------------------------------------------- |
| `tenants`  | Top-level billing/auth boundary                             |
| `users`    | Member of a tenant, with a role                             |
| `events`   | Each captured prompt / response / code_change               |
| `findings` | Each rule match on an event                                 |
| `policies` | Per-tenant action overrides (allow / warn / redact / block) |

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

The tenant tables (`events`, `findings`, `policies`) declare `ENABLE ROW LEVEL SECURITY` and a single permissive policy keyed on `current_setting('app.tenant_id', true)` in the Postgres dialect. The SQLite tables have no RLS declarations (single-tenant local mode).

The generated migration (`drizzle/postgres/0000_*.sql`) contains:

- `ALTER TABLE "events" ENABLE ROW LEVEL SECURITY;`
- `CREATE POLICY "events_tenant_isolation" ... USING ("tenant_id" = current_setting('app.tenant_id', true))`
- (same for `findings` and `policies`)

**FORCE ROW LEVEL SECURITY companion:** drizzle-kit 0.31 emits `ENABLE ROW LEVEL SECURITY` and `CREATE POLICY` but NOT `FORCE ROW LEVEL SECURITY`. Without FORCE, the table owner role bypasses RLS entirely. FORCE is added by a **journaled custom migration** `drizzle/postgres/0001_force_rls.sql`, created with `drizzle-kit generate --custom --name force_rls` so it is recorded in `meta/_journal.json` (idx 1) and applied by the standard drizzle-orm migrator after `0000_initial`. It is the only hand-written SQL in the project. (Note: a plain file sorted by name — e.g. a `9999_*.sql` — would NOT be applied, because the migrator reads the journal, not the directory listing.) When drizzle-kit gains native FORCE RLS support, this migration should be removed and `0000_initial` regenerated.

**Per-request tenant context.** `withTenantContext` opens a transaction and issues `set_config('app.tenant_id', $1, true)` (tenantId bound, never interpolated) so Postgres FORCE RLS resolves the active tenant; SQLite is a passthrough. Routes never call it directly. A **tenant-scope plugin** (`apps/backend/src/plugins/tenant-scope.ts`) decorates each request with `req.tenantScope` — an `open` `TenantScope` bound to that request's tenant — and repositories open their own context from it via `runInScope`. Isolation therefore lives in the data layer: a route cannot reach the database without RLS in force, because the only db-bearing value it can touch is the scope (typed `TenantScope | null`, narrowed by `requireTenantScope` which throws loudly rather than allow an un-scoped query). For work that must be atomic across several repository calls, `withTenantTransaction` opens one transaction and yields a `joined` scope shared by every call, so they commit or roll back together.

## Multi-tenancy (stub)

The current auth hook is a single-user bearer token stub. Real multi-tenancy (Better Auth + row-level tenantId filtering via `SET LOCAL app.tenant_id`) is planned for Week 3. The schema and repository layer are already tenancy-aware — every row carries a `tenant_id`, RLS policies are declared in the Postgres dialect, and the FORCE companion migration is in place.

## Follow-ups (known gaps)

**Dialect-explicit schema type aliases:** the former `CatalogSchema`/`TenantSchema` aliases were SQLite-typed only, so a Postgres handle had no type to describe it. They are replaced by dialect-explicit aliases in `@aka/schema/drizzle`: `SqliteSchema`/`PgSchema` (the full combined handles) and the logical seams `SqliteCatalogSchema`, `SqliteTenantSchema`, `PgCatalogSchema`, `PgTenantSchema`. Each is derived with `Pick<...>` over the runtime `sqliteSchema`/`pgSchema` objects, so the types cannot drift from what `drizzle()` actually receives — a misspelled or removed table is a compile error, and new tables are tracked automatically. The dialect is part of the name because a Postgres table is not assignable to a SQLite one. Locked by type-level assertions in `packages/schema/src/drizzle/schema-types.test.ts` (enforced by `tsc --noEmit`).

**Server-derived `findings.tenantId`:** `tenantId` is **server-derived** — injected from the authenticated context, never accepted from a request body. `packages/schema/src/zod/finding.ts` now mirrors the events pattern: the full `Finding` row schema carries `tenantId` (restoring Zod↔Drizzle parity with `findings.tenant_id`), and `DetectedFinding = Finding.omit({ tenantId: true })` is the producer-side shape. The future findings repository injects `tenantId` from the tenant scope on write — exactly as `insertEvents(scope, batch, userId)` reads `scope.tenantId` for `IngestEvent` — giving a single source of truth that lines up with the Postgres RLS `WITH CHECK (tenant_id = current_setting('app.tenant_id'))`. Decision locked by `packages/schema/src/zod/finding.test.ts`.
