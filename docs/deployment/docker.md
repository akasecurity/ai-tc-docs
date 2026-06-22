# Docker Deployment

AKA ships with Docker configurations for both development and production.

## Development stack

`docker/docker-compose.dev.yml` brings up the full developer environment:

```bash
docker compose -f docker/docker-compose.dev.yml up
```

### Services

| Service         | Port        | Description                                  |
| --------------- | ----------- | -------------------------------------------- |
| `postgres`      | 5432        | PostgreSQL 16                                |
| `backend`       | 4000        | Backend in hosted/Postgres mode, live reload |
| `backend-local` | 4001        | Backend in local/SQLite mode, live reload    |
| `dashboard`     | 5173        | Vite dev server with HMR                     |
| `docs`          | 8000        | MkDocs Material live reload                  |
| `mailpit`       | 8025 / 1025 | Email preview (SMTP + web UI)                |

### Selective startup

Run only the services you need:

```bash
# Backend + Postgres only
docker compose -f docker/docker-compose.dev.yml up postgres backend

# Local SQLite mode only (no Postgres)
docker compose -f docker/docker-compose.dev.yml up backend-local

# Docs site only
docker compose -f docker/docker-compose.dev.yml up docs
```

### Development credentials

The dev stack uses fixed credentials for convenience. **Never use these in production.**

| Variable            | Dev value                                   |
| ------------------- | ------------------------------------------- |
| `AKA_LOCAL_TOKEN`   | `devtoken123456789`                         |
| `POSTGRES_PASSWORD` | `akadev`                                    |
| `DATABASE_URL`      | `postgresql://aka:akadev@postgres:5432/aka` |

## Production image

`docker/Dockerfile` is a multi-stage build:

```
base        node:26-slim + pnpm
deps        pnpm install --frozen-lockfile
dashboard-builder   Vite build → apps/backend/public
backend-builder     tsup build → apps/backend/dist
runtime     node:26-slim + only production artifacts
```

Build the image:

```bash
docker build -f docker/Dockerfile . --tag aka-control-plane:latest
```

Run it:

```bash
docker run -d \
  --name aka \
  -p 4000:4000 \
  -e MODE=self-hosted \
  -e STORAGE_DRIVER=postgres \
  -e DATABASE_URL=postgresql://user:pass@db:5432/aka \
  -e AKA_LOCAL_TOKEN=$(openssl rand -hex 32) \
  -e MIGRATE_ON_START=true \
  aka-control-plane:latest
```

The container serves both the API and the dashboard from port 4000:

- `http://localhost:4000/` — Dashboard SPA
- `http://localhost:4000/healthz` — Health check
- `http://localhost:4000/v1/events` — Events API

## Production docker-compose

`docker/docker-compose.yml` is the production shape — a single `app` service with an optional `postgres` profile:

```bash
# SQLite (simplest self-hosted)
docker compose -f docker/docker-compose.yml up

# With Postgres
docker compose -f docker/docker-compose.yml --profile postgres up
```

### Environment file

Create a `.env` file for production secrets (do not commit this):

```bash
# .env (gitignored)
MODE=self-hosted
STORAGE_DRIVER=postgres
DATABASE_URL=postgresql://aka:supersecret@db:5432/aka
AKA_LOCAL_TOKEN=<output of: openssl rand -hex 32>
MIGRATE_ON_START=true
VERSION=1.0.0
```

Then:

```bash
docker compose --env-file .env -f docker/docker-compose.yml up -d
```

## Health checks

The production container's Docker `HEALTHCHECK` targets the **readiness** probe, so the container reports unhealthy when its database is unreachable (not merely when the process is up):

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||4000)+'/healthz/ready').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
```

```bash
docker inspect aka --format '{{.State.Health.Status}}'
# → healthy
```

Three probes are available (all unauthenticated): `/healthz` (overall), `/healthz/live` (liveness — map a Kubernetes `livenessProbe` here), and `/healthz/ready` (readiness, `503` when a dependency is down — map a `readinessProbe` here). See the [API reference](../api/reference.md).

## Observability (OpenTelemetry)

Telemetry is **off by default**. The collector, Jaeger, and Prometheus are an opt-in compose profile, so a plain `docker compose up` adds no extra containers and no overhead:

```bash
OTEL_ENABLED=true docker compose -f docker/docker-compose.yml --profile observability up
# Jaeger → http://localhost:16686 ; Prometheus → http://localhost:9090
```

To ship to a managed backend (AWS CloudWatch/X-Ray, Azure Monitor) instead, edit `docker/otel-collector-config.yaml` and provide vendor credentials to the collector — no app rebuild. Full guide: [Observability](../operations/observability.md).

## Persistent data

For SQLite in production, mount a volume for the database file:

```bash
docker run -d \
  -v aka-data:/data \
  -e SQLITE_PATH=/data/aka.db \
  aka-control-plane:latest
```

For Postgres, use a managed database (AWS RDS, Supabase, Neon, etc.) and set `DATABASE_URL` accordingly.

## Updating

Rolling update with zero downtime requires Postgres (SQLite is single-writer):

```bash
# Pull new image
docker pull aka-control-plane:latest

# Restart with migration
docker stop aka
docker rm aka
docker run -d \
  --name aka \
  -e MIGRATE_ON_START=true \
  ... (same env as before) ...
  aka-control-plane:latest
```

Migrations are idempotent — re-running an applied migration is safe.

## Building for multiple architectures

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f docker/Dockerfile \
  --tag aka-control-plane:latest \
  --push \
  .
```

## CI/CD

The GitHub Actions workflow (`ci.yml`) runs `docker build` on every push to verify the image builds. Add a push step for your registry:

```yaml
- name: Push image
  run: |
    docker tag aka-control-plane:latest ghcr.io/your-org/aka-control-plane:${{ github.sha }}
    docker push ghcr.io/your-org/aka-control-plane:${{ github.sha }}
```
