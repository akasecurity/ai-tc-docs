# Dev Mode

Dev mode runs AKA with the full multi-tenant identity stack: **Better Auth
sessions**, real tenant provisioning, and **Postgres** storage. Use it when you
need to exercise the dashboard login/sign-up flow or the session-based API auth
that production (`hosted` / `self-hosted`) uses.

> **Dev mode requires Postgres.** SQLite is reserved for `local` and `test`. The
> config loader picks its schema by `STORAGE_DRIVER`, and the
> `local`-token contract makes `MODE=dev` + `STORAGE_DRIVER=sqlite` an
> unsatisfiable combination — the backend refuses to boot. Use `local` mode for
> a no-Docker, no-Postgres setup (see [Local Mode](local.md)).

## Local vs Dev at a glance

| Aspect               | Local mode                       | Dev mode                                 |
| -------------------- | -------------------------------- | ---------------------------------------- |
| Storage              | SQLite                           | Postgres                                 |
| API auth (`/v1`)     | Bearer `AKA_LOCAL_TOKEN`         | Better Auth **session**                  |
| Identity             | Seeded stub tenant/user          | Real sign-up + auto-provisioned tenant   |
| `AKA_LOCAL_TOKEN`    | Required (16+ chars)             | **Must NOT be set** (rejected by config) |
| `BETTER_AUTH_SECRET` | Optional                         | **Required** (32+ chars)                 |
| Migrations on boot   | `MIGRATE_ON_START=true` (SQLite) | Run manually (`migrate:pg`)              |

## 1. Start Postgres

The dev compose file ships a ready-to-use Postgres service (`aka` / `akadev`):

```bash
docker compose -f docker/docker-compose.dev.yml up -d postgres
```

This exposes Postgres on `localhost:5432` with database `aka`.

## 2. Apply migrations

`MIGRATE_ON_START` only runs the SQLite migrator, so Postgres migrations are
applied explicitly:

```bash
ADMIN_DATABASE_URL=postgresql://aka:akadev@localhost:5432/aka \
  pnpm --filter @aka/backend migrate:pg
```

Re-running is safe — the migrator skips already-applied migrations.

## 3. Start the backend

```bash
MODE=dev \
  STORAGE_DRIVER=postgres \
  DATABASE_URL=postgresql://aka:akadev@localhost:5432/aka \
  BETTER_AUTH_SECRET=$(openssl rand -hex 32) \
  BETTER_AUTH_URL=http://localhost:4000 \
  pnpm --filter @aka/backend dev
```

Notes:

- **No `AKA_LOCAL_TOKEN`.** The config loader rejects it in non-local modes — dev
  authenticates via Better Auth, not a shared Bearer token.
- `BETTER_AUTH_SECRET` must be at least 32 characters. Setting `BETTER_AUTH_URL`
  also silences the "Base URL is not set" warning.
- A successful boot logs `{"mode":"dev","storageDriver":"postgres"}` on
  `/healthz`.

## 4. Start the dashboard

```bash
pnpm --filter @aka/dashboard dev
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

## Environment variable quick reference

```bash
MODE=dev                              # Required: enables the Better Auth session path
STORAGE_DRIVER=postgres               # Required: dev mode is Postgres-only
DATABASE_URL=postgres://…/aka         # Required: runtime (app) connection
ADMIN_DATABASE_URL=postgres://…/aka   # Migrations (owner role)
BETTER_AUTH_SECRET=<32+ chars>        # Required: signs sessions
BETTER_AUTH_URL=http://localhost:4000 # Recommended: base URL for callbacks
PORT=4000                             # HTTP port (default: 4000)
```
