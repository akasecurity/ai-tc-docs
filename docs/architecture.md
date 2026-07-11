# Architecture

## What it does

```
Developer ──► Claude Code Plugin ──► AKA AI-TC ──► AI Model
                                            │
                                    Detection Engine
                                    (scan / redact)
                                            │
                                    Policy Resolution
                                    (allow / warn / redact / block)
                                            │
                                    Local Store / Dashboard
                                    (audit log, findings, alerts)
```

Every prompt, tool call, and response that flows through a Claude session is
captured by the plugin, scanned against your rule packs, and logged — before
the AI sees it or after it responds.

## The OSS stack at a glance

The stack is a TypeScript-strict pnpm monorepo:

| Package                   | Role                                                                |
| -------------------------- | ------------------------------------------------------------------- |
| `apps/plugin-claude-code` | Claude Code hooks — captures prompts/responses, fail-open           |
| `apps/web-ui`             | OSS Next.js dashboard — reads the local SQLite store directly       |
| `apps/cli`                | The `aka` CLI — local scan, stats, dashboard, plugin management     |
| `packages/detections`     | Pure detection engine — scan + redact, no I/O                       |
| `packages/schema`         | Zod contracts — single source of truth for all API shapes           |
| `packages/persistence`    | Local SQLite store adapter — no backend required                    |
| `packages/plugin-sdk`     | Shared adapter interface + runtime lifecycle                        |
| `rules/`                  | Detection rule packs (core-pii, secrets)                            |

See the [System Overview](architecture/overview.md) for the full picture:
package dependency rules, the local-first plugin data flow, and API client
generation. For the enterprise backend's architecture (Postgres schema,
Row-Level Security, request lifecycle), see the enterprise docs.
