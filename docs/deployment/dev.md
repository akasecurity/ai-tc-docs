# Dev Mode

Dev mode runs AKA with the full multi-tenant identity stack: **Better Auth
sessions**, real tenant provisioning, and **Postgres** storage. Use it when you
need to exercise the dashboard login/sign-up flow or the session-based API auth
that production (`hosted` / `self-hosted`) uses.

> **The enterprise backend is Postgres-only.** The SQLite storage path was removed
> from the enterprise API, so set `STORAGE_DRIVER=postgres` for every non-`test`
> mode — the backend has no SQLite repositories to fall back on. For a no-Docker,
> no-Postgres setup, run the OSS stack instead (the `aka` CLI + web-ui read the
> local store directly — see [Local / Single-Node](local.md)).

## Local vs Dev at a glance

| Aspect               | Local mode                       | Dev mode                                        |
| -------------------- | -------------------------------- | ----------------------------------------------- |
| Storage              | SQLite                           | Postgres                                        |
| API auth (`/v1`)     | Bearer `AKA_LOCAL_TOKEN`         | Better Auth **session**                         |
| Identity             | Seeded stub tenant/user          | Real sign-up + auto-provisioned tenant          |
| `AKA_LOCAL_TOKEN`    | Required (16+ chars)             | **Must NOT be set** (rejected by config)        |
| `BETTER_AUTH_SECRET` | Optional                         | **Required** (32+ chars)                        |
| Migrations on boot   | `MIGRATE_ON_START=true` (SQLite) | Auto-run by `dev` when `ADMIN_DATABASE_URL` set |

## 1. Start Postgres

The dev compose file ships a ready-to-use Postgres service (`aka` / `akadev`):

```bash
docker compose -f docker/docker-compose.dev.yml up -d postgres
```

This exposes Postgres on `localhost:5432` with database `aka`.

## 2. Apply migrations

> **Optional.** The `dev` script (step 3) applies pending Postgres migrations
> automatically when `ADMIN_DATABASE_URL` is set. Run this step manually only
> when you want to migrate without booting the backend (e.g. CI).

`MIGRATE_ON_START` only runs the SQLite migrator (the runtime app connects as a
non-owner role and must never hold DDL privileges), so Postgres migrations run
as a separate **owner-DSN** step:

```bash
ADMIN_DATABASE_URL=postgresql://aka:akadev@localhost:5432/aka \
  pnpm --filter @alsoknownassecurity/backend migrate:pg
```

Re-running is safe — the migrator skips already-applied migrations.

## 3. Start the backend

```bash
MODE=dev \
  STORAGE_DRIVER=postgres \
  DATABASE_URL=postgresql://aka_app:akaapppw@localhost:5432/aka \
  ADMIN_DATABASE_URL=postgresql://aka:akadev@localhost:5432/aka \
  BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
  BETTER_AUTH_URL=http://localhost:4000 \
  pnpm --filter @alsoknownassecurity/backend dev
```

Notes:

- **Two roles, two DSNs.** `DATABASE_URL` is the non-owner app role
  (`aka_app`, `NOSUPERUSER NOBYPASSRLS`) so FORCE RLS is enforced exactly as in
  production — matching the compose backend. `ADMIN_DATABASE_URL` is the owner
  role (`aka`), used only for the migration step below; the runtime never needs
  it.
- **Migrations auto-apply, and boot fails fast if they error.** With
  `ADMIN_DATABASE_URL` set, `dev` runs the owner-DSN Postgres migrator (a
  separate step, not the runtime process) before starting — so pulling a change
  that adds a table can't leave you on a stale schema. It's chained with `&&`, so
  a migration failure aborts the boot before `tsx watch` starts — intentional (a
  half-applied schema is worse than a broken-looking server). If `pnpm dev` exits
  without serving, read the migrator output just above the exit. Omit
  `ADMIN_DATABASE_URL` to skip auto-migrate (e.g. you migrate out of band).
- **No `AKA_LOCAL_TOKEN`.** The config loader rejects it in non-local modes — dev
  authenticates via Better Auth, not a shared Bearer token.
- `BETTER_AUTH_SECRET` must be at least 32 characters. Setting `BETTER_AUTH_URL`
  also silences the "Base URL is not set" warning.
- A successful boot logs `{"mode":"dev","storageDriver":"postgres"}` on
  `/healthz`.

## 4. Start the dashboard

```bash
pnpm --filter @alsoknownassecurity/dashboard dev
```

Navigate to `http://localhost:5173`. The Vite dev server proxies `/v1`,
`/healthz`, and `/api/auth` to the backend on `localhost:4000`. The backend
trusts the `:5173` dev origin automatically in `local`/`dev` modes (production
serves the dashboard same-origin, so it needs no extra trust).

## 5. Sign up

There is no seeded login in dev mode. On the login page choose **Create one**,
then sign up with any email and a password of **8+ characters**. Sign-up:

1. Creates a Better Auth user (credential + session cookie).
2. Auto-provisions a tenant and a catalog user row for that account.
3. Drops you into the dashboard.

From then on, every `/v1` request is authorized by the session cookie and scoped
to the provisioned tenant. A request without a valid session returns `401`.

## Fully Dockerized alternative

To run the entire dev stack (Postgres + migrations + backend) in containers
instead of on the host, use the full compose file — see [Docker](docker.md):

```bash
docker compose -f docker/docker-compose.dev.yml up
```

To also bring up tracing (Jaeger) and metrics (Prometheus), add the opt-in
observability profile and enable OpenTelemetry:

```bash
OTEL_ENABLED=true docker compose -f docker/docker-compose.dev.yml --profile observability up
# Jaeger → http://localhost:16686 ; Prometheus → http://localhost:9090
```

See [Observability](../operations/observability.md) for the full guide.

## Environment variable quick reference

```bash
MODE=dev                              # Required: enables the Better Auth session path
STORAGE_DRIVER=postgres               # Required: dev mode is Postgres-only
DATABASE_URL=postgres://aka_app:…/aka # Required: runtime — non-owner app role (RLS enforced)
ADMIN_DATABASE_URL=postgres://aka:…/aka # Migrations only — owner role
BETTER_AUTH_SECRET=<32+ chars>        # Required: signs sessions
BETTER_AUTH_URL=http://localhost:4000 # Recommended: base URL for callbacks
PORT=4000                             # HTTP port (default: 4000)
```
