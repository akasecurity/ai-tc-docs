# AKA AI Control Plane

**AKA** is an open-source control plane that sits between your developers and their AI tools — intercepting prompts, scanning for sensitive data, enforcing policy, and giving security teams full visibility into what's being shared with AI models.

## What it does

```
Developer ──► Claude Code Plugin ──► AKA Control Plane ──► AI Model
                                            │
                                    Detection Engine
                                    (scan / redact)
                                            │
                                    Policy Resolution
                                    (allow / warn / redact / block)
                                            │
                                    Backend + Dashboard
                                    (audit log, findings, alerts)
```

Every prompt, tool call, and response that flows through a Claude session is captured by the plugin, scanned against your rule packs, and logged — before the AI sees it or after it responds.

## OSS vs. enterprise

This site documents the **open-source surface**: the Claude Code plugin, the
`aka` CLI, the local SQLite store, the OSS web dashboard, and the detection
rule format. It runs entirely on your machine with no backend, no Postgres,
and no account.

AKA also has an **enterprise control plane** (`apps/backend` + `apps/dashboard`)
for teams — hosted or self-hosted, Postgres-backed, with multi-tenant policy
management and a REST API. That surface is documented separately in the
enterprise docs, not here.

## Key concepts

| Term          | Description                                                                      |
| ------------- | -------------------------------------------------------------------------------- |
| **Event**     | A prompt, response, or code change captured from a Claude session                |
| **Finding**   | A rule match produced by the detection engine against an event                   |
| **Rule**      | A JSON file describing what to detect: keyword list, regex pattern, or validator |
| **Rule Pack** | A directory of rules + mandatory fixtures with a `manifest.json`                 |
| **Policy**    | A per-tenant decision: what action to take when a rule or category fires         |
| **Plugin**    | The Claude Code extension (hooks) that intercepts sessions                       |

## Architecture at a glance

The stack is a TypeScript-strict pnpm monorepo:

| Package                   | Role                                                                |
| ------------------------- | ------------------------------------------------------------------- |
| `apps/backend`            | Fastify 5 API — ingest events, store findings, serve policy bundles |
| `apps/dashboard`          | React + TanStack Query UI — events table, findings, policy editor   |
| `apps/plugin-claude-code` | Claude Code hooks — captures prompts/responses, fail-open           |
| `packages/detections`     | Pure detection engine — scan + redact, no I/O                       |
| `packages/schema`         | Zod contracts — single source of truth for all API shapes           |
| `packages/config`         | Env loader — the only package allowed to read `process.env`         |
| `packages/client`         | Typed fetch client — the only package allowed to call `fetch()`     |
| `packages/plugin-sdk`     | Shared adapter interface + runtime lifecycle                        |
| `rules/`                  | Detection rule packs (core-pii, secrets)                            |

## Quick start

The plugin + `aka` CLI write a local SQLite store (`~/.aka/data/aka.db`) and
the OSS web-ui reads it directly — no backend, no Postgres, no account.

```bash
git clone https://github.com/your-org/ai-control-plane
cd ai-control-plane
pnpm setup
pnpm --filter @alsoknownassecurity/cli build

# `aka` ships as a published package; from a source checkout, run the built bin:
pnpm --filter @alsoknownassecurity/cli exec aka dashboard   # OSS web-ui over ~/.aka/data
```

See the [Quickstart](getting-started/quickstart.md) for a complete walkthrough.

## Project status

AKA is in **active alpha**. The schema contracts, hook interface, and rule format are stable enough to build on. Better Auth integration, hosted tenancy, and the offline queue are on the roadmap.

!!! warning "Alpha software"
Breaking changes may occur between releases. Pin the version in production.
