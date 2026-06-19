# API Reference

All API routes are served by `apps/backend` on port 4000 (default). The current auth scheme is a Bearer token that must match `AKA_LOCAL_TOKEN`.

## Interactive docs (OpenAPI)

The backend generates an OpenAPI 3.1 spec directly from the `@aka/schema` Zod
contracts — the same schemas used to validate requests and serialize responses,
so the spec never drifts from the running code.

- **Swagger UI:** `GET /docs` — browse and try every endpoint.
- **Raw spec:** `GET /openapi.json` — feed to client/codegen tooling.

Both are public (no Bearer token required). Each schema appears once under
`components/schemas` named after its TypeScript type (e.g. `Event`, `IngestBatch`,
`PolicyBundle`) and is referenced by `$ref`.

## Authentication

Every request (except `GET /healthz`, `GET /docs`, and `GET /openapi.json`) must include:

```
Authorization: Bearer <AKA_LOCAL_TOKEN>
```

All errors share one envelope. Missing or incorrect tokens return:

```json
HTTP 401
{ "error": { "code": "UNAUTHORIZED", "message": "Unauthorized" } }
```

Request validation failures return `400` with `code: "VALIDATION_ERROR"` and a
`details` array of the offending Zod issues.

---

## GET /healthz

Health check endpoint. No authentication required.

**Response `200 OK`:**

```json
{
  "status": "ok",
  "mode": "local",
  "storageDriver": "sqlite",
  "version": "0.0.1"
}
```

**Example:**

```bash
curl http://localhost:4000/healthz
```

---

## POST /v1/events

Ingest a batch of events. Deduplicates by `event.id` — re-sending the same ID is safe and returns `duplicates: 1`.

**Request body:**

```json
{
  "events": [
    {
      "id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "sourceTool": "claude-code",
      "kind": "prompt",
      "occurredAt": "2026-06-10T20:00:00Z",
      "contentHash": "sha256-of-content",
      "content": "The full prompt text here",
      "metadata": {
        "sessionId": "session-123",
        "model": "claude-sonnet-4-6"
      }
    }
  ]
}
```

**Event fields:**

| Field                | Type     | Required | Description                                   |
| -------------------- | -------- | -------- | --------------------------------------------- |
| `id`                 | UUID     | Yes      | Client-generated idempotency key              |
| `sourceTool`         | string   | Yes      | `"claude-code"`, `"cursor"`, etc.             |
| `kind`               | enum     | Yes      | `"prompt"` \| `"response"` \| `"code_change"` |
| `occurredAt`         | ISO 8601 | Yes      | When the event happened                       |
| `contentHash`        | string   | Yes      | Hash of the content for deduplication         |
| `content`            | string   | Yes      | Full text of the prompt/response              |
| `metadata`           | object   | No       | Arbitrary key-value pairs                     |
| `metadata.sessionId` | string   | No       | Claude session identifier                     |
| `metadata.model`     | string   | No       | AI model used                                 |
| `metadata.repo`      | string   | No       | Repository name                               |
| `metadata.filePath`  | string   | No       | File being edited                             |

**Response `200 OK`:**

```json
{
  "accepted": 1,
  "duplicates": 0
}
```

**Example:**

```bash
curl -X POST http://localhost:4000/v1/events \
  -H "Authorization: Bearer mytoken1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "events": [{
      "id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "sourceTool": "claude-code",
      "kind": "prompt",
      "occurredAt": "2026-06-10T20:00:00Z",
      "contentHash": "abc123",
      "content": "Help me write a function"
    }]
  }'
```

**Validation error `400 Bad Request`:**

```json
{
  "error": {
    "formErrors": [],
    "fieldErrors": {
      "events": ["Invalid uuid"]
    }
  }
}
```

---

## GET /v1/events

List ingested events, most recent first.

**Query parameters:**

| Parameter | Type    | Default | Description                                |
| --------- | ------- | ------- | ------------------------------------------ |
| `limit`   | integer | `50`    | Max events to return (1–200)               |
| `cursor`  | string  | —       | Pagination cursor from a previous response |

**Response `200 OK`:**

```json
{
  "items": [
    {
      "id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "tenantId": "00000000-0000-0000-0000-000000000001",
      "userId": "00000000-0000-0000-0000-000000000002",
      "sourceTool": "claude-code",
      "kind": "prompt",
      "occurredAt": "2026-06-10T20:00:00Z",
      "contentHash": "abc123",
      "content": "Help me write a function",
      "metadata": null
    }
  ],
  "nextCursor": null
}
```

**Example:**

```bash
curl "http://localhost:4000/v1/events?limit=10" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/policy-bundle

Returns the policy bundle for the authenticated tenant. The Claude Code plugin caches this bundle and uses it for in-process policy resolution.

**Response `200 OK`:**

```json
{
  "policies": [
    {
      "ruleId": "core-pii/email",
      "action": "warn"
    },
    {
      "category": "secret",
      "action": "block"
    }
  ],
  "customKeywords": [],
  "bundleVersion": "2026-06-10T00:00:00Z"
}
```

The current implementation returns the default policy bundle (`DEFAULT_ACTIONS` from `@aka/schema`). A full policy editor is on the roadmap.

**Example:**

```bash
curl http://localhost:4000/v1/policy-bundle \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## POST /v1/policies

Creates a policy for the authenticated tenant. The body is a `CreatePolicyRequest`
(a `Policy` without `id`/`tenantId` — both are server-assigned). At most one policy
may exist per `(scope, target)` within a tenant.

**Request body:**

```json
{
  "scope": "repo",
  "target": { "category": "secret" },
  "action": "block",
  "enabled": true
}
```

**Response `200 OK`** — the created policy, including its generated `id` and `tenantId`.

**Response `409 Conflict`** — a policy with the same `(scope, target)` already exists
for this tenant:

```json
{
  "error": {
    "code": "CONFLICT",
    "message": "A policy with the same (scope, target) already exists for this tenant"
  }
}
```

**Example:**

```bash
curl -X POST http://localhost:4000/v1/policies \
  -H "Authorization: Bearer mytoken1234567890" \
  -H "Content-Type: application/json" \
  -d '{"scope":"repo","target":{"category":"secret"},"action":"block","enabled":true}'
```

---

## POST /v1/rules/test

Runs **draft** detection rules through the engine against ad-hoc text and/or
fixtures, without publishing or installing anything. Use it to preview and tune a
matcher while authoring a rule. The response is stateless — nothing is persisted.

The request body must contain `rules` plus at least one of `text` or a non-empty
`fixtures` array (otherwise there is nothing to test → `400`).

**Request body:**

```json
{
  "rules": [
    {
      "specVersion": 1,
      "id": "draft/email",
      "name": "Email address",
      "category": "pii",
      "severity": "low",
      "matcher": {
        "type": "regex",
        "pattern": "[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}",
        "flags": "gi"
      }
    }
  ],
  "text": "mail me at jane@example.com",
  "fixtures": [
    { "label": "has email", "text": "a@b.com", "shouldMatch": true },
    { "label": "no email", "text": "just words", "shouldMatch": false }
  ]
}
```

**Body fields:**

| Field      | Type            | Required | Description                                                |
| ---------- | --------------- | -------- | ---------------------------------------------------------- |
| `rules`    | `Rule[]`        | Yes      | Draft rules to evaluate (1–100), same shape as a pack rule |
| `text`     | string          | No\*     | Ad-hoc text to scan; returns matches with no pass/fail     |
| `fixtures` | `RuleFixture[]` | No\*     | Test cases (`{ label, text, shouldMatch }`), 0–200         |

\* At least one of `text` / `fixtures` must be present.

**Response `200 OK`:**

```json
{
  "adhoc": {
    "matches": [
      {
        "ruleId": "draft/email",
        "category": "pii",
        "severity": "low",
        "span": { "start": 11, "end": 27 },
        "confidence": 0.9,
        "match": "jane@example.com"
      }
    ]
  },
  "fixtures": [
    {
      "label": "has email",
      "shouldMatch": true,
      "didMatch": true,
      "passed": true,
      "matches": [
        /* ... */
      ]
    },
    { "label": "no email", "shouldMatch": false, "didMatch": false, "passed": true, "matches": [] }
  ],
  "summary": { "total": 2, "passed": 2, "failed": 0 },
  "unsupportedRuleIds": []
}
```

- `adhoc` is present only when `text` was supplied.
- `match` is the **raw** matched substring (unlike findings, which mask) — the
  input is your own test text, so this is what lets you tune the matcher.
- `unsupportedRuleIds` lists rules whose matcher type the engine cannot evaluate
  yet (e.g. `validator`); they silently never match, so they are flagged here.

**Example:**

```bash
curl -X POST http://localhost:4000/v1/rules/test \
  -H "Authorization: Bearer mytoken1234567890" \
  -H "Content-Type: application/json" \
  -d '{
    "rules": [{
      "specVersion": 1, "id": "draft/email", "name": "Email",
      "category": "pii", "severity": "low",
      "matcher": { "type": "regex", "pattern": "[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}" }
    }],
    "text": "mail me at jane@example.com"
  }'
```

---

## Error responses

All error responses follow the same shape:

| Status | Body                                                                      | When                         |
| ------ | ------------------------------------------------------------------------- | ---------------------------- |
| `400`  | `{"error": {"formErrors": [...], "fieldErrors": {...}}}`                  | Zod validation failed        |
| `401`  | `{"error": "Unauthorized"}`                                               | Missing/invalid Bearer token |
| `500`  | `{"statusCode": 500, "error": "Internal Server Error", "message": "..."}` | Unhandled exception          |

---

## Planned endpoints

These endpoints are defined in `packages/schema/src/zod/api.ts` but not yet implemented:

| Method | Path           | Description                          |
| ------ | -------------- | ------------------------------------ |
| `GET`  | `/v1/findings` | List detection findings with filters |
| `PUT`  | `/v1/policies` | Update policy bundle                 |

Track progress in the GitHub issues.
