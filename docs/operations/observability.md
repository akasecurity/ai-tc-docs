# Observability

AKA is instrumented with [OpenTelemetry](https://opentelemetry.io) for **distributed tracing**, **metrics**, and **trace-correlated logs**, following CNCF conventions. Telemetry is **off by default** and is a first-class opt-in: when `OTEL_ENABLED=false` (the default) the SDK is never loaded, no collector is required, and there is no per-request overhead — ideal for single-user / local deployments.

## Architecture

Every service emits **OTLP** to an **OpenTelemetry Collector**. The collector is the single egress that fans telemetry out to backends — Jaeger and Prometheus by default, or a managed vendor (AWS CloudWatch/X-Ray, Azure Monitor) by collector config alone. Application code only ever speaks OTLP, so switching vendors never requires an app rebuild.

```
 plugin ─(x-request-id + traceparent)─┐
 dashboard (web SDK, fetch) ──────────┤
                                       ▼
        backend / registry (Fastify)  ──OTLP──►  OpenTelemetry Collector
        • auto: http, fastify, pg                 ├─► Jaeger        (traces)
        • manual: db (sqlite+pg), auth            ├─► Prometheus    (metrics)
        • /metrics, /healthz/live, /healthz/ready ├─► CloudWatch/X-Ray (optional)
                                                  └─► Azure Monitor   (optional)
```

## What is instrumented

- **HTTP, Fastify, and Postgres** via OpenTelemetry auto-instrumentation.
- **Database queries** (both SQLite and Postgres) via an explicit `db.query` span + duration metric at the single tenant-scope chokepoint every repository call flows through — SQLite has no auto-instrumentation, so this is its only DB visibility.
- **Auth** — `verifyApiKey`, `getSession`, and tenant resolution each get a span.
- **Outgoing calls** — `@alsoknownassecurity/client` injects W3C `traceparent`/`tracestate`/`baggage` and a correlation id on every backend→registry and plugin→backend request, so a trace spans the whole chain.

## Correlation id & log correlation

Every request carries a correlation id:

- The backend reuses an inbound `x-request-id`, else the active **trace id**, else a generated UUID, and **echoes it back** on the `x-request-id` response header.
- Every log line is stamped with `trace_id` and `span_id`, so logs and traces line up in your backend of choice.

## Metrics

When enabled, each service exposes a Prometheus endpoint at `:9464/metrics` (configurable via `PROMETHEUS_METRICS_PORT`) **and** pushes metrics over OTLP to the collector. HTTP request duration (from the HTTP auto-instrumentation) and DB operation duration (the custom `db.query` metric) are emitted.

## Health probes (always on)

These probes are always available regardless of `OTEL_ENABLED`. Liveness is
dependency-free; readiness is dependency-aware (it checks the database):

| Endpoint         | Purpose   | Behaviour                                                                                    |
| ---------------- | --------- | -------------------------------------------------------------------------------------------- |
| `/healthz`       | overall   | Static `{ status, mode, storageDriver, version }` (back-compat).                             |
| `/healthz/live`  | liveness  | `200` while the process is up — a failure means "restart me". No dependency checks.          |
| `/healthz/ready` | readiness | `200` only when the database answers `SELECT 1`; **`503`** otherwise, with per-check detail. |

Kubernetes mapping: wire `livenessProbe` → `/healthz/live` and `readinessProbe` → `/healthz/ready`. The Docker image ships a `HEALTHCHECK` against `/healthz/ready`.

## Running the stack

The collector, Jaeger, and Prometheus are an **opt-in compose profile** — a plain `docker compose up` starts only the app.

```bash
# Dev stack with observability
OTEL_ENABLED=true docker compose -f docker/docker-compose.dev.yml --profile observability up

# Self-hosted (production) image with observability
OTEL_ENABLED=true docker compose -f docker/docker-compose.yml --profile observability up
```

| UI / endpoint       | URL                      |
| ------------------- | ------------------------ |
| Jaeger              | <http://localhost:16686> |
| Prometheus          | <http://localhost:9090>  |
| Collector OTLP HTTP | `http://localhost:4318`  |

**Self-hosted exposure.** Jaeger and Prometheus serve **unauthenticated** trace data and metrics, so in the production compose (`docker-compose.yml`) their UIs are bound to **loopback** (`127.0.0.1:16686` / `127.0.0.1:9090`) and the collector publishes no host ports — services reach it over the compose network. For remote access, front them with an **authenticated reverse proxy** (or an SSH tunnel); never publish them on `0.0.0.0`. (The dev compose binds all interfaces for convenience — not for production use.)

## Sending to a managed backend (CloudWatch / Azure)

Because the app only emits OTLP, vendor routing lives entirely in `docker/otel-collector-config.yaml`:

1. Uncomment the `awsemf`/`awsxray` (AWS) or `azuremonitor` (Azure) exporter block.
2. Add the exporter to the relevant `traces` / `metrics` pipeline.
3. Provide credentials to the **collector** container (`AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, or `AZURE_MONITOR_CONNECTION_STRING`) — the production compose file (`docker/docker-compose.yml`) already passes these through.

No application change or rebuild is required.

## Turning it off

Unset `OTEL_ENABLED` (or set it to `false`) and don't start the `observability` profile. The services run exactly as they did before instrumentation — no collector, no SDK, no overhead.
