# Configuration

AKA is configured entirely through environment variables. The `@alsoknownassecurity/config` package validates them at startup — the server crashes with a complete error list if anything is missing or invalid, rather than failing silently later.

## Modes

The `MODE` variable selects which schema is validated and which features are
enabled. The enterprise backend is **Postgres-only**; every non-`test` mode
requires `DATABASE_URL`.

| `MODE`        | Use case                                                    |
| ------------- | ----------------------------------------------------------- |
| `dev`         | Development team, Postgres, full stack                      |
| `hosted`      | Production SaaS multi-tenant (Postgres)                     |
| `self-hosted` | Production single-tenant Docker deployment (Postgres)       |
| `test`        | Vitest — the integration suite runs against a real Postgres |

> Single-user / single-node with no server runs the **OSS stack** — the `aka` CLI
> and web-ui read the local SQLite store via `@alsoknownassecurity/persistence`, not the
> enterprise backend. See [Local / Single-Node](../deployment/local.md).

## Storage driver

The enterprise backend uses Postgres exclusively — Postgres is an infra dependency
it connects to, not a bundled engine. The SQLite storage path was removed from the
enterprise API, so **`STORAGE_DRIVER=postgres` is required** for every non-`test`
mode; the backend has no SQLite repositories to fall back on. (Config-level
validation still defaults `STORAGE_DRIVER` to `sqlite` for the OSS tooling — set it
explicitly to `postgres` when running the enterprise backend.)

| `STORAGE_DRIVER` | Driver               | Required vars  |
| ---------------- | -------------------- | -------------- |
| `postgres`       | node-postgres (`pg`) | `DATABASE_URL` |

(SQLite is used only by the OSS local store — `@alsoknownassecurity/persistence`, Node's
`node:sqlite` — and by the rule registry service; neither is the enterprise API.)

## All variables

### Common (all modes)

| Variable           | Default      | Description                                                        |
| ------------------ | ------------ | ------------------------------------------------------------------ |
| `NODE_ENV`         | `production` | Node environment: `development`, `test`, `production`              |
| `MODE`             | `local`      | One of: `local` (default), `dev`, `hosted`, `self-hosted`, `test`  |
| `STORAGE_DRIVER`   | —            | **Required: `postgres`** for the enterprise backend                |
| `PORT`             | `4000`       | HTTP port the backend listens on                                   |
| `HOST`             | `0.0.0.0`    | Interface to bind                                                  |
| `LOG_LEVEL`        | `info`       | Pino log level: `fatal`, `error`, `warn`, `info`, `debug`, `trace` |
| `MIGRATE_ON_START` | `false`      | Run pending migrations before accepting traffic                    |
| `VERSION`          | `0.0.0`      | Injected by CI into the Docker image; appears in `/healthz`        |

### Test mode

| Variable          | Default | Description                                                                                                                                                                                   |
| ----------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AKA_LOCAL_TOKEN` | —       | **Required for `MODE=test`.** Bearer token (min 16 chars) used as the auth token in the test harness. Not used or accepted for `dev`/`hosted`/`self-hosted` — those use Better Auth API keys. |

### Postgres mode

| Variable             | Default | Description                                                                                                                                                                                            |
| -------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `DATABASE_URL`       | —       | **Required.** PostgreSQL connection string. Must target the non-superuser runtime role `aka_app` (see [Postgres roles](#postgres-roles) below). Example: `postgresql://aka_app:akaapppw@host:5432/aka` |
| `BETTER_AUTH_SECRET` | —       | **Required for `MODE=dev`/`hosted`/`self-hosted`.** Secret used by Better Auth to sign session tokens (min 32 chars). Generate with `openssl rand -hex 32`.                                            |
| `BETTER_AUTH_URL`    | —       | **Required for `MODE=dev`/`hosted`/`self-hosted`.** Public base URL of the backend (e.g. `https://api.yourhost.com`). Used by Better Auth for redirect/cookie issuing.                                 |

### Better Auth (authentication server)

AKA uses [Better Auth](https://better-auth.com) for identity in non-local modes. Users sign up with email + password; machine clients (the Claude Code plugin) authenticate with an API key sent in the `x-api-key` header.

`AKA_LOCAL_TOKEN` must **not** be set for `MODE=dev|hosted|self-hosted`. Setting it will cause a startup validation failure. It is only valid for `local`/`test` modes where Better Auth is not used.

### Observability (OpenTelemetry)

OpenTelemetry is **off by default** and carries zero overhead when disabled — the SDK is never loaded and no telemetry code runs. Set `OTEL_ENABLED=true` to turn on distributed tracing, metrics, and trace-correlated logs. See [Observability](../operations/observability.md) for the full guide.

| Variable                      | Default                      | Description                                                                                |
| ----------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------ |
| `OTEL_ENABLED`                | `false`                      | Master switch. When `false`, the OTel SDK is never started — zero overhead.                |
| `OTEL_SERVICE_NAME`           | per-service                  | `service.name` resource attribute. Defaults to `aka-backend` / `aka-registry`.             |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://otel-collector:4318` | Base OTLP endpoint of the collector. Apps emit OTLP; the collector fans out to backends.   |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf`              | OTLP wire protocol. Only `http/protobuf` is supported.                                     |
| `OTEL_TRACES_SAMPLER_ARG`     | `1.0`                        | Parent-based trace sampling ratio in `[0,1]` (`1.0` = sample everything).                  |
| `OTEL_METRICS_EXPORTER`       | `otlp,prometheus`            | Comma list of metric readers: `otlp` (push to collector) and/or `prometheus` (pull).       |
| `PROMETHEUS_METRICS_PORT`     | `9464`                       | Port the Prometheus exporter self-serves `/metrics` on (when `prometheus` is in the list). |
| `OTEL_LOG_LEVEL`              | `info`                       | OTel internal diagnostic log level: `none`, `error`, `warn`, `info`, `debug`, `verbose`.   |

The dashboard (browser tracing) uses the build-time equivalents `VITE_OTEL_ENABLED` and `VITE_OTEL_EXPORTER_OTLP_ENDPOINT`; when `VITE_OTEL_ENABLED` is unset, the OpenTelemetry web SDK is dead-code-eliminated from the bundle.

### Postgres roles

AKA uses two distinct Postgres roles to keep schema ownership separate from runtime access and to ensure `FORCE ROW LEVEL SECURITY` is never bypassed.

| Role      | Type                                      | Purpose                                                                                                                                                                             |
| --------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aka`     | Owner (superuser in the Docker image)     | Runs DDL and migrations (`CREATE TABLE`, `ALTER TABLE`, etc.). Because Postgres superusers bypass `FORCE RLS` unconditionally, this role is **never** used for application queries. |
| `aka_app` | Non-superuser (`NOSUPERUSER NOBYPASSRLS`) | Runtime role the application and integration tests connect as. Subject to all RLS policies. `DATABASE_URL` must point at this role.                                                 |

**Why two roles?** Postgres superusers bypass `FORCE ROW LEVEL SECURITY` regardless of the policy definition. If the application connected as `aka` (the owner/superuser created by the Docker image), RLS would enforce nothing and rows from one tenant would be visible to any other. `aka_app` — with `NOSUPERUSER NOBYPASSRLS` — is the only role for which the `app.tenant_id` GUC-based policies actually fire.

**Local dev and CI**: the file `docker/postgres-init/01-app-role.sql` is mounted into `/docker-entrypoint-initdb.d/` of the Postgres container. It runs once on first init, creating `aka_app` and setting default privileges so every table `aka` creates later is automatically accessible to `aka_app`.

**Existing volumes**: init scripts in `/docker-entrypoint-initdb.d/` only run when the data directory is **empty**. If you ran the dev stack before this role existed, your `postgres-dev-data` volume already has data, so the script is skipped and you will see `role "aka_app" does not exist`. Recreate the volume to pick it up with `docker compose -f docker/docker-compose.dev.yml down -v` (this deletes local dev data), then start the stack again.

**Hosted deployments**: `aka_app`'s password (`akaapppw`) is for local dev and CI only. In production, inject the credentials via your managed secret provider and set `DATABASE_URL` accordingly — never commit production credentials.

### Plugin configuration (local-first)

The Claude Code plugin is **not** configured through environment variables — hooks
are bare processes Claude Code spawns, so env can't reach them. It owns a local
store and settings under `~/.aka` instead:

- `~/.aka/data/aka.db` — the SQLite store the plugin writes (events + findings).
- `~/.aka/settings/settings.json` — onboarding answers (`mode`, `redaction`),
  written by `/aka:setup`. See [Claude Code plugin → Configuration](../plugin/claude-code.md#configuration).

The plugin works with **zero backend**. The optional local backend (below) reads
that same `aka.db`; an enterprise backend URL + token (Phase 2) live in
`~/.aka/settings/config.json`, not in the environment.

## Example configurations

> Single-user with no server? Don't configure the backend at all — run the OSS
> stack (the `aka` CLI + web-ui read `~/.aka/data/aka.db` directly). See
> [Local / Single-Node](../deployment/local.md).

=== "Docker dev (Postgres)"

    ```bash
    MODE=dev
    STORAGE_DRIVER=postgres
    # Runtime role: the non-superuser aka_app so FORCE RLS is active.
    DATABASE_URL=postgresql://aka_app:akaapppw@postgres:5432/aka
    # Owner role: migrations only (MIGRATE_ON_START), never runtime queries.
    ADMIN_DATABASE_URL=postgresql://aka:akadev@postgres:5432/aka
    # Better Auth required for non-local modes — generate a secret with: openssl rand -hex 32
    BETTER_AUTH_SECRET=<generate-with-openssl-rand-hex-32>
    BETTER_AUTH_URL=http://localhost:4000
    PORT=4000
    MIGRATE_ON_START=true
    LOG_LEVEL=debug
    ```

=== "Production self-hosted"

    ```bash
    MODE=self-hosted
    STORAGE_DRIVER=postgres
    DATABASE_URL=postgresql://aka_app:<password>@db.internal:5432/aka
    ADMIN_DATABASE_URL=postgresql://aka:<password>@db.internal:5432/aka
    BETTER_AUTH_SECRET=<generate-with-openssl-rand-hex-32>
    BETTER_AUTH_URL=https://api.yourhost.com
    PORT=4000
    MIGRATE_ON_START=true
    LOG_LEVEL=warn
    VERSION=1.2.3
    ```

## Generating a secure secret

```bash
openssl rand -hex 32
# → a3f8c2e7d1b9456f0e2a8c4d6b3e7f1a2c5d8e9b0f3a6c9d2e5f8b1c4d7e0f3
```

Use the output as `BETTER_AUTH_SECRET` (`dev`/`hosted`/`self-hosted`) or `AKA_LOCAL_TOKEN` (`test`).

For local mode, `AKA_LOCAL_TOKEN` must be the same value set in both the backend and the Claude Code plugin environment.

## Validation errors

If required variables are missing or fail validation, the backend exits with a detailed error:

```
Error: Configuration errors (mode=local):
  • AKA_LOCAL_TOKEN: String must contain at least 16 character(s)
  • SQLITE_PATH: Required
```

This is intentional — silent misconfiguration is worse than a loud startup failure.
