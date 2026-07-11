# OSS vs. Enterprise

This site documents the **open-source surface**: the Claude Code / Claude
Desktop plugin, the `aka` CLI, the local SQLite store, the OSS web dashboard,
and the detection rule format. It runs entirely on your machine with no
backend, no Postgres, and no account.

AKA also has an **enterprise control plane** for teams — hosted or
self-hosted, Postgres-backed, with multi-tenant policy management and a REST
API. That surface is documented separately in the enterprise docs, not here.

|                    | OSS                                              | Enterprise                                              |
| ------------------ | ------------------------------------------------- | --------------------------------------------------------- |
| **Storage**        | Local SQLite (`~/.aka/data/aka.db`)              | Postgres                                                 |
| **Backend**        | None — plugin/CLI/web-ui read the store directly | Fastify API (`apps/backend`)                             |
| **Dashboard**      | OSS Next.js web-ui, reads the local store         | React SPA, calls the backend REST API                    |
| **Auth**           | None (single-user, local machine)                | Better Auth — sessions + API keys, multi-tenant           |
| **Policy scope**   | Per-installation                                 | Per-tenant, centrally managed                            |
| **Deployment**     | Nothing to deploy — runs on your machine          | Hosted SaaS or self-hosted Docker/Postgres               |
| **Rule packs**     | Bundled + `aka detections update`                 | Same, plus org-wide rollout via the rule registry         |

Both share the same detection engine (`packages/detections`), the same rule
format, and the same Zod contracts (`packages/schema`) — a rule you write once
runs identically in the plugin, the OSS CLI, and the enterprise backend.

## Attaching a standalone install to the enterprise platform

A standalone plugin install can be pointed at the enterprise backend
(`runMode: attached` / `aka attach`) instead of the local store, for when the
hosted control plane — not the local file — should be the source of truth.
See the [Claude Code](plugin/claude-code.md) or [Claude Desktop](plugin/claude-desktop.md)
plugin guide and the enterprise docs for the full auth handshake and
data-sync model.
