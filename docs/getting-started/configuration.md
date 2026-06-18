# Configuration

AKA is configured entirely through environment variables. The `@aka/config` package validates them at startup — the server crashes with a complete error list if anything is missing or invalid, rather than failing silently later.

## Modes

The `MODE` variable selects which schema is validated and which features are enabled.

| `MODE`        | Use case                                        |
| ------------- | ----------------------------------------------- |
| `local`       | Single developer, SQLite, no auth server needed |
| `dev`         | Development team, Postgres, full stack          |
| `hosted`      | Production SaaS multi-tenant                    |
| `self-hosted` | Production single-tenant Docker deployment      |
| `test`        | Vitest — uses in-memory SQLite, skips migration |

## Storage drivers

| `STORAGE_DRIVER` | Driver               | Required vars  |
| ---------------- | -------------------- | -------------- |
| `sqlite`         | better-sqlite3       | `SQLITE_PATH`  |
| `postgres`       | node-postgres (`pg`) | `DATABASE_URL` |

## All variables

### Common (all modes)

| Variable           | Default      | Description                                                           |
| ------------------ | ------------ | --------------------------------------------------------------------- |
| `NODE_ENV`         | `production` | Node environment: `development`, `test`, `production`                 |
| `MODE`             | —            | **Required.** One of: `local`, `dev`, `hosted`, `self-hosted`, `test` |
| `STORAGE_DRIVER`   | —            | **Required.** One of: `sqlite`, `postgres`                            |
| `PORT`             | `4000`       | HTTP port the backend listens on                                      |
| `HOST`             | `0.0.0.0`    | Interface to bind                                                     |
| `LOG_LEVEL`        | `info`       | Pino log level: `fatal`, `error`, `warn`, `info`, `debug`, `trace`    |
| `MIGRATE_ON_START` | `false`      | Run pending migrations before accepting traffic                       |
| `VERSION`          | `0.0.0`      | Injected by CI into the Docker image; appears in `/healthz`           |

### SQLite mode (local/test only)

| Variable          | Default    | Description                                                                                                                                                                                        |
| ----------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SQLITE_PATH`     | `./aka.db` | Filesystem path to the SQLite database file. Use `:memory:` for tests.                                                                                                                             |
| `AKA_LOCAL_TOKEN` | —          | **Required for `MODE=local` and `MODE=test`.** Bearer token (min 16 chars) used as the auth token. Not used or accepted for `dev`/`hosted`/`self-hosted` modes — use Better Auth API keys instead. |

### Postgres mode

| Variable             | Default | Description                                                                                                                                                                                            |
| -------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `DATABASE_URL`       | —       | **Required.** PostgreSQL connection string. Must target the non-superuser runtime role `aka_app` (see [Postgres roles](#postgres-roles) below). Example: `postgresql://aka_app:akaapppw@host:5432/aka` |
| `BETTER_AUTH_SECRET` | —       | **Required for `MODE=dev`/`hosted`/`self-hosted`.** Secret used by Better Auth to sign session tokens (min 32 chars). Generate with `openssl rand -hex 32`.                                            |
| `BETTER_AUTH_URL`    | —       | **Required for `MODE=dev`/`hosted`/`self-hosted`.** Public base URL of the backend (e.g. `https://api.yourhost.com`). Used by Better Auth for redirect/cookie issuing.                                 |

### Better Auth (authentication server)

AKA uses [Better Auth](https://better-auth.com) for identity in non-local modes. Users sign up with email + password; machine clients (the Claude Code plugin) authenticate with an API key sent in the `x-api-key` header.

`AKA_LOCAL_TOKEN` must **not** be set for `MODE=dev|hosted|self-hosted`. Setting it will cause a startup validation failure. It is only valid for `local`/`test` modes where Better Auth is not used.

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

### Plugin variables

These are read by `apps/plugin-claude-code` (not the backend). Set them in the shell where Claude Code runs.

| Variable          | Description                                                                                                                                                       |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AKA_BACKEND_URL` | Backend base URL. Default: `http://localhost:4000`                                                                                                                |
| `AKA_LOCAL_TOKEN` | **Local mode only.** Must match the backend's `AKA_LOCAL_TOKEN`. In non-local modes, the plugin uses a Better Auth API key instead (configured via `/aka:setup`). |

## Example configurations

=== "Local SQLite"

    ```bash
    MODE=local
    STORAGE_DRIVER=sqlite
    SQLITE_PATH=./aka.db
    AKA_LOCAL_TOKEN=mytoken1234567890
    MIGRATE_ON_START=true
    PORT=4000
    LOG_LEVEL=info
    ```

=== "Docker dev (Postgres)"

    ```bash
    MODE=dev
    STORAGE_DRIVER=postgres
    # Connect as the non-superuser runtime role so FORCE RLS is active.
    # The owner role (aka) is used only for migrations, never for runtime queries.
    DATABASE_URL=postgresql://aka_app:akaapppw@postgres:5432/aka
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
    DATABASE_URL=postgresql://user:secret@db.internal:5432/aka
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

Use the output as `BETTER_AUTH_SECRET` (non-local modes) or `AKA_LOCAL_TOKEN` (local/test modes).

For local mode, `AKA_LOCAL_TOKEN` must be the same value set in both the backend and the Claude Code plugin environment.

## Validation errors

If required variables are missing or fail validation, the backend exits with a detailed error:

```
Error: Configuration errors (mode=local):
  • AKA_LOCAL_TOKEN: String must contain at least 16 character(s)
  • SQLITE_PATH: Required
```

This is intentional — silent misconfiguration is worse than a loud startup failure.
