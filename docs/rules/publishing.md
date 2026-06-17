# Publishing rules to the marketplace

Detection rules are versioned, attributed, and distributed through a **global rule registry** — a
public catalog of rule packs. AKA Labs' base rules (the `rules/` directory in this repo) are
published to it automatically from CI; in later phases, users will publish their own packs and fork
existing ones from the dashboard.

## How distribution works

The registry (`apps/registry`) is its own service with its own database, separate from the
per-customer control plane. It exposes:

| Method & path                                        | Auth            | Purpose                       |
| ---------------------------------------------------- | --------------- | ----------------------------- |
| `GET /v1/packs`                                      | public          | Browse the catalog            |
| `GET /v1/packs/:namespace/:packId`                   | public          | Pack metadata + version list  |
| `GET /v1/packs/:namespace/:packId/versions/:version` | public          | Full pack content (rules)     |
| `POST /v1/packs/:namespace/:packId/versions`         | publisher token | Publish a version             |
| `POST /v1/packs/:namespace/:packId/fork`             | publisher token | Fork a pack (records lineage) |

Each publisher owns a **namespace** (e.g. `aka-labs`). Publishing requires a token scoped to that
namespace, so no one else can publish under the AKA Labs name. Browsing and reading are public.

### Versions are immutable

The registry derives a content hash from a pack's rules + metadata. Publishing a `(pack, version)`:

- **identical content** at an already-published version → no-op (`unchanged`);
- **changed content** at an already-published version → rejected (`409`).

So editing a rule means bumping `version` in its pack's `manifest.json`.

## Publishing the AKA Labs base rules (CI)

The [`release-rules.yml`](https://github.com/alsoknownassecurity/ai-control-plane/blob/main/.github/workflows/release-rules.yml)
workflow publishes everything under `rules/` to the `aka-labs` namespace:

1. Bump `version` in each changed `rules/<pack>/manifest.json`.
2. Push a tag `rules-v<version>` (e.g. `rules-v2026.06.17`).
3. CI runs the detection fixtures (the correctness gate), validates the packs (`--dry-run`), then
   publishes with the `RULES_PUBLISH_TOKEN` secret to `RULES_REGISTRY_URL`.

Run the publisher locally against a dev registry:

```bash
RULES_REGISTRY_URL=http://127.0.0.1:4555 RULES_PUBLISH_TOKEN=<token> \
  pnpm --filter @aka/rules-publisher exec tsx src/cli.ts --rules-dir rules

# validate only, no network/token:
pnpm --filter @aka/rules-publisher exec tsx src/cli.ts --rules-dir rules --dry-run
```

## Running the registry locally

```bash
RULES_PUBLISH_TOKEN=<≥16-char token> SQLITE_PATH=./registry.db PORT=4555 \
  pnpm --filter @aka/registry exec tsx src/main.ts
```

In local/test mode the service seeds the `aka-labs` publisher and a token from `RULES_PUBLISH_TOKEN`
on boot. In production, tokens are issued through an admin flow rather than seeded from the environment.

> **Status:** Phase 1 covers the registry, schema, and CI publishing. Installing packs into a control
> plane (so the plugin enforces them) and authoring/forking from the dashboard are later phases.
