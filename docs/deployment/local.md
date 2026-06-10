# Local Mode

Local mode runs AKA as a single-user, single-process setup using SQLite. It requires no Docker, no Postgres, and no additional services — just Node.js and pnpm.

## Use cases

- Individual developer running the full stack on their laptop
- CI environments running integration tests
- Quick demos and evaluations

## Starting the backend

The built binary is the most portable option:

```bash
# Build first
pnpm build

# Run
MODE=local \
  STORAGE_DRIVER=sqlite \
  AKA_LOCAL_TOKEN=mytoken1234567890 \
  SQLITE_PATH=./aka.db \
  MIGRATE_ON_START=true \
  node apps/backend/dist/main.js
```

For development with live reload:

```bash
MODE=local \
  STORAGE_DRIVER=sqlite \
  AKA_LOCAL_TOKEN=mytoken1234567890 \
  SQLITE_PATH=./aka.db \
  MIGRATE_ON_START=true \
  pnpm --filter @aka/backend dev
```

## What `MIGRATE_ON_START` does

When `MIGRATE_ON_START=true`, the backend:

1. Runs all pending migrations from `apps/backend/drizzle/sqlite/`
2. Creates stub tenant and user rows (`00000000-0000-0000-0000-000000000001/2`) for local single-user mode

These seed rows satisfy the foreign key constraints in the events table. In production multi-tenant mode, real tenant/user rows are created via the auth flow instead.

## SQLite file location

Set `SQLITE_PATH` to any writable path:

| Value                 | Use case                                           |
| --------------------- | -------------------------------------------------- |
| `./aka.db`            | Persistent file in the current directory           |
| `/var/lib/aka/aka.db` | Persistent file in a system directory              |
| `:memory:`            | In-memory only (used by Vitest; data lost on exit) |

## Running the dashboard

The dashboard dev server proxies API calls to the backend:

```bash
pnpm --filter @aka/dashboard dev
```

Navigate to `http://localhost:5173`. The proxy is configured in `apps/dashboard/vite.config.ts` to forward `/v1` and `/healthz` to `localhost:4000`.

## Running the docs

With Python installed:

```bash
pip install -r apps/docs/requirements.txt
pnpm --filter @aka/docs dev
```

Navigate to `http://localhost:8000`.

## Running everything locally (without Docker)

```bash
# Terminal 1 — backend
MODE=local STORAGE_DRIVER=sqlite AKA_LOCAL_TOKEN=mytoken1234567890 \
  MIGRATE_ON_START=true SQLITE_PATH=./aka.db \
  pnpm --filter @aka/backend dev

# Terminal 2 — dashboard
pnpm --filter @aka/dashboard dev

# Terminal 3 — docs (optional)
pnpm --filter @aka/docs dev
```

## Applying migrations manually

To apply migrations without starting the server (useful for CI or scripts):

```bash
# Via the migrator CLI
pnpm --filter @aka/migrator start -- sqlite \
  --db-path ./aka.db \
  --migrations-dir apps/backend/drizzle/sqlite

# Or directly with drizzle-kit
pnpm --filter @aka/backend migrate
```

## Typical local workflow

```bash
# 1. Start the backend with fresh DB
rm -f ./aka.db
MODE=local STORAGE_DRIVER=sqlite AKA_LOCAL_TOKEN=mytoken1234567890 \
  MIGRATE_ON_START=true pnpm --filter @aka/backend dev &

# 2. Verify it's up
curl http://localhost:4000/healthz

# 3. Start a Claude Code session — the plugin will forward events
# (make sure AKA_BACKEND_URL and AKA_LOCAL_TOKEN are exported)

# 4. Watch events appear
watch -n2 'curl -s http://localhost:4000/v1/events \
  -H "Authorization: Bearer mytoken1234567890" | jq .items[].id'
```

## Environment variable quick reference

```bash
MODE=local                        # Required: enables local mode
STORAGE_DRIVER=sqlite             # Required: use SQLite
AKA_LOCAL_TOKEN=<16+ chars>      # Required: Bearer token for API access
SQLITE_PATH=./aka.db              # Where to store the database
MIGRATE_ON_START=true             # Run migrations + seed on startup
PORT=4000                         # HTTP port (default: 4000)
LOG_LEVEL=info                    # Logging verbosity
```
