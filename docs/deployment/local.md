# Local / Single-Node

There are two ways to run AKA locally, depending on whether you want the
enterprise control plane or just the open-source single-node experience.

## Option A — OSS single-node (no server, no Docker, no Postgres)

The open-source surface needs **no backend at all**. The Claude Code plugin (and
the `aka` CLI) write a local SQLite store at `~/.aka/data/aka.db` via
`@alsoknownassecurity/persistence` (Node's built-in `node:sqlite` — no native dependency), and
the OSS web dashboard reads that same store directly in Server Components. Nothing
leaves the machine; there is no account, no network, no database server.

```bash
# The plugin self-installs and writes ~/.aka/data/aka.db during Claude Code sessions.
# Browse your local findings/health with the OSS dashboard:
aka dashboard        # launches the Next.js web-ui over ~/.aka/data
# or a terminal view:
aka tui
```

See the [CLI guide](../getting-started/cli.md) and the
[Claude Code plugin](../plugin/claude-code.md) page. This is the recommended
single-user local experience.

## Option B — the enterprise backend, locally

The enterprise control plane (`apps/backend`) is **Postgres-only** — Postgres is
an infra dependency it connects to, never a bundled engine. Run it against the
Postgres that `docker-compose` provides:

```bash
docker compose -f docker/docker-compose.dev.yml up
```

This brings up Postgres (with the `aka_app` runtime role seeded from
`docker/postgres-init/`), the backend, and the enterprise dashboard. The backend
reads `DATABASE_URL` (the non-superuser `aka_app` role) at runtime and applies
migrations at boot via `ADMIN_DATABASE_URL` (the owner role) when
`MIGRATE_ON_START=true`.

To run the backend directly against a Postgres you provide:

```bash
MODE=dev \
  STORAGE_DRIVER=postgres \
  DATABASE_URL=postgresql://aka_app:akaapppw@localhost:5432/aka \
  ADMIN_DATABASE_URL=postgresql://aka:akadev@localhost:5432/aka \
  MIGRATE_ON_START=true \
  BETTER_AUTH_SECRET="$(openssl rand -hex 32)" \
  BETTER_AUTH_URL=http://localhost:4000 \
  pnpm --filter @alsoknownassecurity/backend dev
```

> The enterprise backend no longer supports SQLite — the SQLite storage path was
> removed from the enterprise API, so set `STORAGE_DRIVER=postgres` for every
> non-`test` mode (the backend has no SQLite repositories to fall back on).

## Applying migrations manually

Apply the Postgres schema migrations via the backend's `migrate:pg` script (it
runs the migrator through `tsx`):

```bash
ADMIN_DATABASE_URL="$ADMIN_DATABASE_URL" \
  pnpm --filter @alsoknownassecurity/backend migrate:pg
```

Upgrading an existing self-hosted install that still runs on the old SQLite image?
Run the one-shot data migration first (the `aka-migrate` bin ships with the
published package; from a repo checkout run its source through `tsx`):

```bash
pnpm exec tsx tools/migrator/src/cli.ts data-sqlite-to-postgres \
  --db-path /app/data/aka.db \
  --url "$ADMIN_DATABASE_URL"
```

## Running the docs

```bash
pip install -r apps/docs/requirements.txt
pnpm --filter @alsoknownassecurity/docs dev
```

Navigate to `http://localhost:8000`.
