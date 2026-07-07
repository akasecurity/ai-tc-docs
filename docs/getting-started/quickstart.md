# Quickstart

This guide takes you from zero to a working AKA installation with the Claude Code plugin active, events being captured, and findings appearing in the dashboard.

## Prerequisites

- **Node.js 26+** and **pnpm 10+**
- **Claude Code** installed and signed in
- **Docker** (optional — needed for the Postgres dev stack)

## 1. Clone and install

```bash
git clone https://github.com/your-org/ai-control-plane
cd ai-control-plane
pnpm setup   # pnpm install + lefthook hooks
```

## 2. Start the backend

The enterprise backend is **Postgres-only**. Bring up the full dev stack (Postgres
is provided as a service):

```bash
docker compose -f docker/docker-compose.dev.yml up
```

Services:

| URL                     | Service                 |
| ----------------------- | ----------------------- |
| `http://localhost:4000` | Backend (Postgres)      |
| `http://localhost:5173` | Dashboard               |
| `http://localhost:8000` | Docs                    |
| `http://localhost:8025` | Mailpit (email preview) |

> For a single-user, no-server experience, use the OSS stack instead — the `aka`
> CLI + web-ui read the local SQLite store directly. See
> [Local / Single-Node](../deployment/local.md).

## 3. Verify the backend is running

```bash
curl http://localhost:4000/healthz
# → {"status":"ok","mode":"dev","storageDriver":"postgres","version":"0.0.1"}
```

## 4. Send a test event

```bash
curl -X POST http://localhost:4000/v1/events \
  -H "Authorization: Bearer mytoken1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "events": [{
      "id": "aaaaaaaa-0000-0000-0000-000000000001",
      "sourceTool": "claude-code",
      "kind": "prompt",
      "occurredAt": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'",
      "contentHash": "abc123",
      "content": "My AWS key is AKIAIOSFODNN7EXAMPLE, can you check this config?"
    }]
  }'
# → {"accepted":1,"duplicates":0}
```

List it back:

```bash
curl http://localhost:4000/v1/events \
  -H "Authorization: Bearer mytoken1234567890"
# → {"items":[{...}],"nextCursor":null}
```

## 5. Install the Claude Code plugin

From the repo root, register the plugin with Claude Code:

```bash
claude mcp add aka-control-plane
# or point Claude Code to the plugin directory:
# Settings → Plugins → Add local plugin → apps/plugin-claude-code
```

The plugin reads two environment variables:

```bash
export AKA_BACKEND_URL=http://localhost:4000
export AKA_LOCAL_TOKEN=mytoken1234567890
```

Add these to your shell profile so they're available in every Claude session.

## 6. Test the plugin hook

Run the hook script directly with a sample payload to verify it works:

```bash
echo '{"prompt":"My SSN is 123-45-6789 and AWS key AKIAIOSFODNN7EXAMPLE"}' \
  | AKA_BACKEND_URL=http://localhost:4000 \
    AKA_LOCAL_TOKEN=mytoken1234567890 \
    node apps/plugin-claude-code/scripts/user-prompt-submit.js
# → {"action":"allow","prompt":"My SSN is 123-45-6789 and AWS key AKIAIOSFODNN7EXAMPLE"}
```

The plugin currently allows all events (warn-only mode). Redaction and blocking are configured via Policy once the full policy editor is implemented.

## 7. Open the dashboard

Navigate to `http://localhost:5173` to see the Events page. The event you sent in step 4 should appear in the table.

## What's next

- [Configure the backend](configuration.md) for your environment
- [Write your first detection rule](../rules/writing-rules.md)
- [Install the plugin for your team](../plugin/claude-code.md)
