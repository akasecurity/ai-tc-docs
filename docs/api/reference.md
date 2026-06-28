# API Reference

All API routes are served by `apps/backend` on port 4000 (default).

## Interactive docs (OpenAPI)

The backend generates an OpenAPI 3.1 spec directly from the `@aka/schema` Zod
contracts — the same schemas used to validate requests and serialize responses,
so the spec never drifts from the running code.

- **Swagger UI:** `GET /docs` — browse and try every endpoint.
- **Raw spec:** `GET /openapi.json` — feed to client/codegen tooling.

Both are public (no auth required). Each schema appears once under
`components/schemas` named after its TypeScript type (e.g. `Event`, `IngestBatch`,
`PolicyBundle`) and is referenced by `$ref`.

The **rule registry** (`apps/registry`) generates its spec the same way and serves
its own **`GET /docs`** and **`GET /openapi.json`**. Browse/read endpoints are
public; only `publishPackVersion` / `forkPack` require a bearer token (marked with
`bearerAuth` in the spec).

### Generated client

`@aka/client` is **generated** from these two specs with
[Hey API](https://heyapi.dev) (`@hey-api/openapi-ts`) — not hand-written. The
pipeline is Zod (`@aka/schema`) → Fastify OpenAPI spec → typed client, so the
client cannot drift from the server. To regenerate after changing a route or
schema:

```bash
pnpm --filter @aka/client gen:openapi          # dump both specs + regenerate
pnpm --filter @aka/client gen:openapi:check     # CI: regenerate and fail on drift
```

The dumped specs (`packages/client/openapi/*.json`) and the generated output
(`packages/client/src/generated/**`) are committed; CI runs the drift check.

## Authentication

AKA supports two auth schemes depending on `MODE`:

### Local / test mode (`MODE=local|test`)

Uses a shared Bearer token that must match `AKA_LOCAL_TOKEN`:

```
Authorization: Bearer <AKA_LOCAL_TOKEN>
```

### Non-local modes (`MODE=dev|hosted|self-hosted`)

Two schemes are supported, checked in this order on each request:

**1. API key (machine clients — Claude Code plugin)**

Send the API key in the `x-api-key` header:

```
x-api-key: <your-api-key>
```

The key is hashed before storage — it is shown **once** at creation. A user must have a valid session to mint a key. See [Minting an API key](#minting-an-api-key) below.

**2. Session cookie (browser / interactive)**

After signing in via `POST /api/auth/sign-in/email`, Better Auth sets a session cookie that is sent automatically with subsequent requests.

---

### Minting an API key

API keys tie a machine client to a specific user's tenant. The flow:

```bash
# 1. Sign up (or sign in if you already have an account)
curl -s -c cookies.txt -X POST http://localhost:4000/api/auth/sign-up/email \
  -H 'content-type: application/json' \
  -d '{"email":"you@example.com","password":"Password123!","name":"Your Name"}'

# 2. Mint an API key (requires the session cookie from step 1)
curl -s -b cookies.txt -X POST http://localhost:4000/api/auth/api-key/create \
  -H 'content-type: application/json' \
  -d '{}'
# → {"key":"<plaintext-key>", "id":"...", "referenceId":"<userId>", ...}
# The plaintext key is shown ONLY ONCE — save it now.

# 3. Use the key for machine requests
curl -H 'x-api-key: <plaintext-key>' http://localhost:4000/v1/events
```

Additional key management endpoints (all require a valid session):

| Method | Path                       | Description                               |
| ------ | -------------------------- | ----------------------------------------- |
| `POST` | `/api/auth/api-key/create` | Mint a new key for the authenticated user |
| `GET`  | `/api/auth/api-key/list`   | List all keys for the authenticated user  |
| `POST` | `/api/auth/api-key/delete` | Revoke a key by `keyId`                   |

---

All errors share one envelope. Missing or incorrect credentials return:

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

## GET /healthz/live

Liveness probe. No authentication required. Returns `200` while the process is up and responsive — no dependency checks (a failure means "restart me"). Wire a Kubernetes `livenessProbe` here.

**Response `200 OK`:**

```json
{ "status": "alive", "uptimeSeconds": 1234 }
```

---

## GET /healthz/ready

Readiness probe. No authentication required. Returns `200` only when every hard dependency is reachable (the database answers `SELECT 1`); otherwise **`503`** with per-check detail. Wire a Kubernetes `readinessProbe` here. The Docker image's `HEALTHCHECK` targets this endpoint.

**Response `200 OK`:**

```json
{ "status": "ready", "checks": [{ "name": "database", "status": "pass", "latencyMs": 1 }] }
```

**Response `503 Service Unavailable`:**

```json
{
  "status": "not_ready",
  "checks": [
    { "name": "database", "status": "fail", "latencyMs": 30, "detail": "connection refused" }
  ]
}
```

---

## GET /metrics

Prometheus metrics endpoint, served on a dedicated port (`PROMETHEUS_METRICS_PORT`, default `9464`) only when `OTEL_ENABLED=true` and `OTEL_METRICS_EXPORTER` includes `prometheus`. See [Observability](../operations/observability.md).

```bash
curl http://localhost:9464/metrics
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

## POST /v1/inventory

Idempotent upsert of a session's **inventory dimensions** (the [Meta] Data Model). The body is an `InventoryContext` — the machine/repo facts the plugin resolves (`host`, `harness`, `project`). The **account** (User/Account) dimension is _not_ accepted from the client; it is derived server-side from the authenticated user. Content-addressed, tenant-scoped ids are computed server-side and returned, so repeat sessions on the same machine/project collapse to one row each.

**Request body** (all members optional):

```json
{
  "host": {
    "objectType": "host",
    "identityKey": "stable-machine-id",
    "title": "my-laptop",
    "attributes": {
      "host_name": "my-laptop",
      "os": "darwin",
      "os_version": "25.5.0",
      "arch": "arm64"
    }
  },
  "harness": {
    "objectType": "harness",
    "identityKey": "claude-code",
    "title": "Claude Code",
    "attributes": { "harness_version": "1.2.3", "interface": "cli" }
  },
  "project": { "url": "git@github.com:org/repo.git", "name": "repo", "attributes": {} }
}
```

Only the `identityKey` (per `objectType`) is hashed into the content-addressed id; everything else rides in the mutable `attributes` bag (Type-1 / overwrite-to-latest, with `first_seen` pinned).

**Response `200 OK`** — the resolved ids, ready to stamp onto a Session audit row:

```json
{ "hostId": "…", "harnessId": "…", "accountId": "…", "sourceProjectId": "…" }
```

---

## POST /v1/audit-events

Append one **audit-event fact** — e.g. the Session root opened at `SessionStart`. Idempotent on the event `id` (re-sending the same id is a no-op). The `user_id` is server-derived from auth; the inventory FKs (`hostId`/`harnessId`/`sourceProjectId`) are the ids returned by `POST /v1/inventory`.

**Request body:**

```json
{
  "id": "claude-session-id",
  "eventType": "session",
  "startedAt": "2026-06-26T20:00:00Z",
  "hostId": "…",
  "harnessId": "…",
  "sourceProjectId": "…",
  "attributes": { "os_version": "25.5.0", "harness_version": "1.2.3" }
}
```

**Response `200 OK`:** `{ "accepted": true }`

---

## GET /v1/facets

Distinct filter-facet values for the authenticated tenant, read from the small inventory dimension (never by scanning the audit fact table). Powers the host/harness/project/os filters on the dashboard read surfaces.

**Response `200 OK`:**

```json
{
  "hosts": ["my-laptop"],
  "harnesses": ["Claude Code"],
  "osVersions": ["25.5.0"],
  "projects": ["repo"]
}
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

## GET /v1/security/findings/severity-summary

Findings grouped by severity, for the **Security** dashboard "Open by severity"
donut. Not range-scoped — reflects findings for the tenant at request time.
**Note:** findings have no resolution state yet, so this counts _all_ findings
(the "open" framing arrives when a finding lifecycle exists). All four severity
levels are always present (count may be 0).

**Query parameters:** none.

**Response `200 OK`:**

```json
{
  "total": 131,
  "bySeverity": [
    { "severity": "critical", "count": 9 },
    { "severity": "high", "count": 23 },
    { "severity": "medium", "count": 41 },
    { "severity": "low", "count": 58 }
  ]
}
```

**Example:**

```bash
curl "http://localhost:4000/v1/security/findings/severity-summary" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## Shared `range` query parameter

The range-driven Security widgets accept a `range` query parameter:

| Parameter | Type | Default | Allowed              |
| --------- | ---- | ------- | -------------------- |
| `range`   | enum | `30d`   | `7d` `30d` `3m` `6m` |

An unsupported value fails validation → `400` with `code: "VALIDATION_ERROR"`.

---

## GET /v1/security/enforcement-actions

Intercepted actions in the window for the **Security** dashboard "Enforcement
actions" card — total plus per-kind counts (`blocked`/`redacted`/`warned`) with
the period-over-period delta vs the immediately preceding window of equal length.

**Query parameters:** `range` (see above).

**Response `200 OK`:**

```json
{
  "range": "30d",
  "total": 1817,
  "actions": [
    { "kind": "blocked", "count": 96, "delta": 12 },
    { "kind": "redacted", "count": 1284, "delta": 148 },
    { "kind": "warned", "count": 437, "delta": -9 }
  ]
}
```

All three kinds are always present (count may be 0). `delta` is signed. Findings
with `actionTaken` of `allow`/`log` are not enforcement and are excluded.

**Example:**

```bash
curl "http://localhost:4000/v1/security/enforcement-actions?range=30d" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/security/findings/timeseries

New detections per time bucket, split by severity, for the **Security** dashboard
"Findings over time" chart. Granularity is server-chosen from the range
(`7d`/`30d` → `day`; `3m`/`6m` → `week`). Buckets with no findings are present
with zeros, ordered oldest → newest.

**Query parameters:** `range` (see above).

**Response `200 OK`:**

```json
{
  "range": "30d",
  "granularity": "day",
  "points": [
    { "timestamp": "2026-05-21", "critical": 7, "high": 19, "medium": 22 },
    { "timestamp": "2026-05-22", "critical": 5, "high": 17, "medium": 25 }
  ]
}
```

The chart plots `critical`/`high`/`medium`; `low` is intentionally omitted.

Buckets are **rolling**, not calendar-aligned: for `week` granularity they tile
forward in 7-day steps from the window start (the day `range` days ago), so a
`timestamp` is not necessarily a Monday.

> **Note — this total will not equal `enforcement-actions` `total`.** The two
> widgets use different windows on purpose: `enforcement-actions` is a rolling
> `[now − range, now)`, while the timeseries is day-aligned to UTC midnight and
> excludes `low` (and counts detections, not enforcement actions). Summing the
> timeseries buckets is expected to differ from the enforcement total.

**Example:**

```bash
curl "http://localhost:4000/v1/security/findings/timeseries?range=30d" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/security/top-sources

Repos and people ranked by findings in the window, for the **Security** dashboard
"Top sources" list. `repo` sources come from the event's `metadata.repo`; `user`
sources are named by the user's email. Sorted by `findingsCount` descending.

**Query parameters:**

| Parameter | Type    | Default | Notes                            |
| --------- | ------- | ------- | -------------------------------- |
| `range`   | enum    | `30d`   | See shared range param above.    |
| `limit`   | integer | `5`     | `1`–`50`.                        |
| `kind`    | enum    | —       | `repo` or `user`; omit for both. |

**Response `200 OK`:**

```json
{
  "range": "30d",
  "items": [
    { "id": "repo_payments-api", "name": "payments-api", "kind": "repo", "findingsCount": 38 },
    { "id": "user_maya-chen", "name": "maya@example.com", "kind": "user", "findingsCount": 21 }
  ]
}
```

`id` is stable for linking to the source's detail view. A `user` source with no
catalog email falls back to the user id as its `name`.

**Example:**

```bash
curl "http://localhost:4000/v1/security/top-sources?range=30d&limit=5" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/security/scan-coverage

Per-provider scan coverage for the **Security** dashboard "Scan coverage" card.
In the initial release only Claude Code is scanned; other providers return
`supported: false` (the FE greys them out / shows "Soon").

**Query parameters:** `range` (see above) — accepted and echoed, but coverage is
currently a constant business fact, not a per-window metric.

**Response `200 OK`:**

```json
{
  "range": "30d",
  "providers": [
    { "provider": "claudecode", "coverage": 100, "supported": true },
    { "provider": "cursor", "coverage": 0, "supported": false },
    { "provider": "chatgpt", "coverage": 0, "supported": false },
    { "provider": "copilot", "coverage": 0, "supported": false },
    { "provider": "api", "coverage": 0, "supported": false }
  ]
}
```

`provider` is a stable id (note `claudecode`, distinct from the event
`sourceTool` `claude-code`). `coverage` is `0` whenever `supported` is `false`.

**Example:**

```bash
curl "http://localhost:4000/v1/security/scan-coverage?range=30d" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/security/recommended-actions

Prioritized, environment-specific suggestions for the **Security** dashboard
"Recommended actions" list. Excludes dismissed/applied (only `open` are returned).

> **Status:** the recommendation **engine** that generates these rows, and the
> executor that performs the underlying change on apply, are not built yet. Rows
> are created via the dev seed; `apply` transitions status and echoes the action
> but does not yet perform the concrete mutation.

**Query parameters:**

| Parameter | Type    | Default | Notes     |
| --------- | ------- | ------- | --------- |
| `limit`   | integer | `3`     | `1`–`20`. |

**Response `200 OK`:**

```json
{
  "items": [
    {
      "id": "rec_01HZX",
      "category": "block_credentials",
      "severity": "critical",
      "title": "Block AWS keys in payments-api",
      "description": "5 access keys reached Claude Code prompts this week. Promote the policy from warn to block for this repo.",
      "subjects": [
        { "type": "repo", "id": "payments-api", "label": "payments-api" },
        { "type": "policy", "id": "pol_8821", "label": "Block cloud credentials" }
      ],
      "action": {
        "mode": "apply",
        "type": "promote_policy_to_block",
        "label": "Promote to block",
        "policyId": "pol_8821"
      }
    }
  ]
}
```

`action.mode` is `apply` (mutation via the apply endpoint) or `navigate` (the FE
deep-links `action.href`, no server call). `category` and `action.type` are
extensible.

---

## POST /v1/security/recommended-actions/{id}/apply

Approve a recommendation: resolves it (drops it off the list) and echoes the
action that was applied. Only valid when `action.mode === "apply"`.

- **Path params:** `id` — the recommendation id.
- **Body:** none.

**Response `200 OK`:**

```json
{
  "id": "rec_01HZX",
  "status": "applied",
  "appliedAction": { "type": "promote_policy_to_block", "policyId": "pol_8821" }
}
```

**Errors:**

| Status | `code`                     | When                                               |
| ------ | -------------------------- | -------------------------------------------------- |
| `404`  | `recommendation_not_found` | Unknown id, or not visible to the caller's tenant. |
| `409`  | `already_resolved`         | Already applied or dismissed.                      |
| `422`  | `not_applicable`           | `action.mode` is `navigate` (nothing to apply).    |

## POST /v1/security/recommended-actions/{id}/dismiss

Dismiss a recommendation (drops it off the list).

- **Path params:** `id`. **Body:** none.
- **Response `200 OK`:** `{ "id": "rec_01HZX", "status": "dismissed" }`
- **Errors:** `404 recommendation_not_found`; `409 already_resolved`.

---

## Error responses

All error responses follow the same shape:

| Status | Body                                                                      | When                         |
| ------ | ------------------------------------------------------------------------- | ---------------------------- |
| `400`  | `{"error": {"formErrors": [...], "fieldErrors": {...}}}`                  | Zod validation failed        |
| `401`  | `{"error": "Unauthorized"}`                                               | Missing/invalid Bearer token |
| `500`  | `{"statusCode": 500, "error": "Internal Server Error", "message": "..."}` | Unhandled exception          |

**Error code convention:**

- **`UPPER_SNAKE`** — framework / Zod validation codes (`VALIDATION_ERROR`, `UNAUTHORIZED`, `NOT_FOUND`, etc.) emitted by the shared error handler.
- **`lower_snake`** — domain-specific codes emitted explicitly by services/routes (`installed_pack_not_found`, `detection_not_found`, `invalid_detection_id`, `already_imported`, `already_up_to_date`). New detections endpoints use this convention.

Both are wrapped in the same envelope: `{ "error": { "code": "...", "message": "..." } }`.

---

## PATCH /v1/installed-packs/:namespace/:packId

Toggle the `enabled` flag on an installed detection pack. Requires an authenticated session.

**Path params:**

| Param       | Description                            |
| ----------- | -------------------------------------- |
| `namespace` | Pack publisher namespace (e.g. `aka`)  |
| `packId`    | Pack identifier (e.g. `cloud-secrets`) |

**Request body:**

```json
{ "enabled": true }
```

At least one field must be present — an empty body `{}` returns `400 VALIDATION_ERROR`.

**Response `200 OK`** — the updated `InstalledPack` resource:

```json
{
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "namespace": "aka",
  "packId": "cloud-secrets",
  "version": "2.3.1",
  "name": "Cloud Credentials",
  "enabled": true
}
```

**Errors:**

| Status | `code`                     | Convention  | When                                             |
| ------ | -------------------------- | ----------- | ------------------------------------------------ |
| `400`  | `VALIDATION_ERROR`         | UPPER_SNAKE | Body is empty or does not match the schema (Zod) |
| `401`  | `UNAUTHORIZED`             | UPPER_SNAKE | No valid auth token                              |
| `404`  | `installed_pack_not_found` | lower_snake | No installed pack with that namespace/packId     |

> **Permission note:** this endpoint does NOT currently return `403 Forbidden`. The codebase has no read/write permission layer — all authenticated tenant users are full-access. Do not build frontend logic against a `403` response; it will not fire.

**Example:**

```bash
curl -X PATCH "http://localhost:4000/v1/installed-packs/aka/cloud-secrets" \
  -H "Authorization: Bearer mytoken1234567890" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}'
```

---

## GET /v1/detections/stats

Returns aggregate statistics for the authenticated tenant's installed detections.

- `detections` — total installed pack count.
- `rules` — sum of rule counts from the `rulesJson` snapshot across all packs.
- `active` — count of installed packs where `enabled = true`.
- `findingsLast30d` — findings produced by the tenant's installed rule-id set in the trailing 30 days.

**Response `200 OK`:**

```json
{
  "detections": 6,
  "rules": 12,
  "active": 5,
  "findingsLast30d": 1123
}
```

When no packs are installed all values are `0`.

**Errors:**

| Status | `code`         | Convention  | When                |
| ------ | -------------- | ----------- | ------------------- |
| `401`  | `UNAUTHORIZED` | UPPER_SNAKE | No valid auth token |

> **Permission note:** this endpoint does NOT return `403 Forbidden` — no read/write permission layer exists yet. All authenticated tenant users are full-access.

**Example:**

```bash
curl "http://localhost:4000/v1/detections/stats" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/detections

List installed detections with optional filter and search.

**Query parameters:**

| Parameter | Type   | Default | Description                                                                                |
| --------- | ------ | ------- | ------------------------------------------------------------------------------------------ |
| `filter`  | enum   | `all`   | `all` \| `library` \| `custom` \| `customized` \| `updates`                                |
| `q`       | string | —       | Case-insensitive substring search over `name`, `packId`, and `namespace` (not description) |

**Sorting:** enabled packs first, then alphabetically by name ascending.

**Fixed values in v1:**

- `origin` is always `"library"` (no custom-authoring model).
- `counts.customized` and `counts.updates` are always `0` (branching model deferred; update-available is lazy on detail only).
- `filter=customized` and `filter=updates` always return an empty `items` array.

**Response `200 OK`:**

```json
{
  "counts": {
    "all": 4,
    "library": 4,
    "custom": 0,
    "customized": 0,
    "updates": 0
  },
  "items": [
    {
      "id": "aka/cloud-secrets",
      "name": "Cloud Credentials",
      "version": "2.3.1",
      "enabled": true,
      "origin": "library",
      "namespace": "aka",
      "packId": "cloud-secrets",
      "ruleCount": 3
    }
  ]
}
```

`items[].id` is the **un-encoded** `namespace/packId` slug. Clients `encodeURIComponent(id)` when addressing the detail or update endpoint (PR-2).

`counts` are computed over the **unfiltered** set — `q` narrows `items` only, not the counts.

**Errors:**

| Status | `code`         | Convention  | When                |
| ------ | -------------- | ----------- | ------------------- |
| `401`  | `UNAUTHORIZED` | UPPER_SNAKE | No valid auth token |

> **Permission note:** this endpoint does NOT return `403 Forbidden` — no read/write permission layer exists yet. All authenticated tenant users are full-access.

**Example:**

```bash
curl "http://localhost:4000/v1/detections?q=cloud" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/detections/:id

Return full detail for a single installed detection. `:id` is the **URL-encoded** `namespace%2FpackId` slug (e.g. `aka%2Fcloud-secrets`).

The list endpoint emits `items[].id` as the un-encoded `namespace/packId` string. Clients encode it via `encodeURIComponent(id)` when calling this endpoint.

**Path params:**

| Param | Description                                                     |
| ----- | --------------------------------------------------------------- |
| `:id` | URL-encoded `namespace/packId` slug, e.g. `aka%2Fcloud-secrets` |

**Behaviour:**

- Rules are served from the **local `rulesJson` snapshot** — authoritative, never blocked by registry downtime.
- `update` is computed lazily by calling the registry. If the registry is unreachable, `update` degrades to `null` (honestly unknown) and the response is still `200`.
- `findingsLast30d` counts findings for this pack's rule-id set over the trailing 30 days.
- Only `regex` matcher type is returned (the engine does not support `hash_index`, `ml_classifier`, or `script`).

**Response `200 OK`:**

```json
{
  "id": "aka/cloud-secrets",
  "name": "Cloud Credentials",
  "version": "2.3.1",
  "enabled": true,
  "origin": "library",
  "namespace": "aka",
  "packId": "cloud-secrets",
  "ruleCount": 3,
  "editedAt": "2026-06-01T00:00:00.000Z",
  "findingsLast30d": 42,
  "latestVersion": "2.5.0",
  "update": { "available": true, "latestVersion": "2.5.0" },
  "rules": [
    {
      "id": "aka/cloud-rule",
      "name": "Cloud Rule",
      "category": "secret",
      "severity": "high",
      "matcher": { "type": "regex", "pattern": "\\bAKIA\\w+", "flags": "g" }
    }
  ],
  "modified": false
}
```

When the registry is unreachable, `"update": null` and `"latestVersion": null`.

**Errors:**

| Status | `code`                 | Convention  | When                                                  |
| ------ | ---------------------- | ----------- | ----------------------------------------------------- |
| `400`  | `invalid_detection_id` | lower_snake | Malformed `:id` — not a valid `namespace/packId` slug |
| `401`  | `UNAUTHORIZED`         | UPPER_SNAKE | No valid auth token                                   |
| `404`  | `detection_not_found`  | lower_snake | No installed pack matching the id                     |

> **Permission note:** this endpoint does NOT return `403 Forbidden` — no read/write permission layer exists yet.

> **Proxy note:** deployments MUST NOT enable `%2F` decoding/normalization (e.g. nginx `merge_slashes`) for `/v1/detections/*`, or the `:id` route will 404 when the slug contains a literal `/`.

**Example:**

```bash
curl "http://localhost:4000/v1/detections/aka%2Fcloud-secrets" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## GET /v1/detections/library

Browse the registry catalog overlaid with the tenant's import state. Returns every pack the registry knows about, annotated with whether the tenant has imported it.

**IMPORTANT:** this route is registered **before** `GET /v1/detections/:id` — `library` is a static path segment, not a slug.

**Query parameters:**

| Parameter  | Type              | Default | Description                                             |
| ---------- | ----------------- | ------- | ------------------------------------------------------- |
| `q`        | string            | —       | Case-insensitive search over name/publisher/description |
| `category` | detectionCategory | —       | Filter to one category (`pii`, `secret`, etc.)          |

**Response `200 OK`:**

```json
{
  "categories": ["secret", "pii"],
  "items": [
    {
      "id": "aka/cloud-secrets",
      "name": "Cloud Credentials",
      "publisher": "aka",
      "version": "2.5.0",
      "ruleCount": 0,
      "description": "Detects cloud credentials",
      "updatedAt": "2025-06-01T00:00:00.000Z",
      "state": "update",
      "importedAs": "aka/cloud-secrets"
    }
  ]
}
```

**`state` values:**

| Value      | Meaning                                         |
| ---------- | ----------------------------------------------- |
| `new`      | Not imported by this tenant                     |
| `imported` | Imported at the current registry version        |
| `update`   | Imported but a newer version exists in registry |

**Known limitation — `ruleCount` is always `0`:** The registry catalog (`PublishedPack`) does not expose a rule count, so `ruleCount` is always `0` in library items. Frontends should treat `0` as "count unavailable" rather than "no rules". This will be resolved once the registry exposes the count in its catalog response (tracked as a registry follow-up).

**Errors:**

| Status | `code`                  | Convention  | When                                    |
| ------ | ----------------------- | ----------- | --------------------------------------- |
| `401`  | `UNAUTHORIZED`          | UPPER_SNAKE | No valid auth token                     |
| `503`  | `registry_unconfigured` | lower_snake | `RULES_REGISTRY_URL` is not set         |
| `502`  | `registry_error`        | lower_snake | Registry is unreachable or returned 5xx |

**Example:**

```bash
curl "http://localhost:4000/v1/detections/library?category=secret" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## POST /v1/detections/import

Import a pack from the registry. Resolves the latest version and installs it as an enabled detection.

**IMPORTANT:** this route is registered **before** `POST /v1/detections/:id/update` — `import` is a static path segment, not a slug.

**Request body:**

```json
{ "libraryId": "aka/cloud-secrets" }
```

`libraryId` must be in `namespace/packId` format — an empty or malformed value returns `400`.

**Response `201 Created`** — the new `DetectionDetail` resource (same shape as `GET /v1/detections/:id`).

**Errors:**

| Status | `code`                  | Convention  | When                                                                            |
| ------ | ----------------------- | ----------- | ------------------------------------------------------------------------------- |
| `400`  | `VALIDATION_ERROR`      | UPPER_SNAKE | `libraryId` missing or malformed                                                |
| `401`  | `UNAUTHORIZED`          | UPPER_SNAKE | No valid auth token                                                             |
| `404`  | `library_not_found`     | lower_snake | Pack does not exist in the registry                                             |
| `409`  | `already_imported`      | lower_snake | Pack already imported; `details.detectionId` contains the existing workspace id |
| `503`  | `registry_unconfigured` | lower_snake | `RULES_REGISTRY_URL` is not set                                                 |
| `502`  | `registry_error`        | lower_snake | Registry unreachable or returned 5xx                                            |

**Example:**

```bash
curl -X POST "http://localhost:4000/v1/detections/import" \
  -H "Authorization: Bearer mytoken1234567890" \
  -H "Content-Type: application/json" \
  -d '{"libraryId": "aka/webhook-tokens"}'
```

---

## POST /v1/detections/:id/update

Pull the latest version of an installed detection from the registry and re-install it. No request body.

**Path params:**

| Param | Description                                                     |
| ----- | --------------------------------------------------------------- |
| `:id` | URL-encoded `namespace/packId` slug, e.g. `aka%2Fcloud-secrets` |

**Behaviour:**

- Fetches `latestVersion` from the registry. If the installed version already matches, returns `409 already_up_to_date`.
- On success, upserts the pack at `latestVersion` and returns the updated `DetectionDetail`.

**Response `200 OK`** — the updated `DetectionDetail` resource (same shape as `GET /v1/detections/:id`).

**Errors:**

| Status | `code`                  | Convention  | When                                                  |
| ------ | ----------------------- | ----------- | ----------------------------------------------------- |
| `400`  | `invalid_detection_id`  | lower_snake | Malformed `:id` — not a valid `namespace/packId` slug |
| `401`  | `UNAUTHORIZED`          | UPPER_SNAKE | No valid auth token                                   |
| `404`  | `detection_not_found`   | lower_snake | No installed pack matching the id                     |
| `409`  | `already_up_to_date`    | lower_snake | Installed version already matches `latestVersion`     |
| `503`  | `registry_unconfigured` | lower_snake | `RULES_REGISTRY_URL` is not set                       |
| `502`  | `registry_error`        | lower_snake | Registry unreachable or returned 5xx                  |

**Example:**

```bash
curl -X POST "http://localhost:4000/v1/detections/aka%2Fcloud-secrets/update" \
  -H "Authorization: Bearer mytoken1234567890"
```

---

## Planned endpoints

These endpoints are defined in `packages/schema/src/zod/api.ts` but not yet implemented:

| Method | Path           | Description                          |
| ------ | -------------- | ------------------------------------ |
| `GET`  | `/v1/findings` | List detection findings with filters |
| `PUT`  | `/v1/policies` | Update policy bundle                 |

Track progress in the GitHub issues.
