# System Overview

## Repository layout

```
ai-control-plane/
├── apps/
│   ├── backend/          Fastify 5 API server
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
│   └── migrator/         CLI for applying DB migrations
├── skills/
│   ├── backend-conventions/SKILL.md
│   └── write-detection-rule/SKILL.md
└── docker/
    ├── Dockerfile
    ├── docker-compose.yml        (prod)
    └── docker-compose.dev.yml    (dev)
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

## Data flow

### Event ingestion (plugin → backend → DB)

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

Five tables, all in SQLite or Postgres (same DDL):

| Table      | Purpose                                                     |
| ---------- | ----------------------------------------------------------- |
| `tenants`  | Top-level billing/auth boundary                             |
| `users`    | Member of a tenant, with a role                             |
| `events`   | Each captured prompt / response / code_change               |
| `findings` | Each rule match on an event                                 |
| `policies` | Per-tenant action overrides (allow / warn / redact / block) |

See `apps/backend/src/db/schema/sqlite.ts` for the full schema.

## Multi-tenancy (stub)

The current auth hook is a single-user bearer token stub. Real multi-tenancy (Better Auth + row-level tenantId filtering) is planned for Week 3. The schema and repository layer are already tenancy-aware — every row carries a `tenantId`.
