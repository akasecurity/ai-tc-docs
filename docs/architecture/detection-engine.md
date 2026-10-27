# Detection Engine

The detection engine lives in `packages/detections`. It is intentionally **pure** — no I/O, no Node-specific APIs, no network calls. Every surface — the plugins' hooks, the `aka` CLI, and the dashboard — runs it in-process.

## Public API

```typescript
import { scan, redact, registerPack, getLoadedRules } from '@akasecurity/detections';

// Scan a text string against all loaded rules (or a specific set).
// context.filePath gates appliesTo.extensions; context.eventKind gates appliesTo.eventKinds.
const findings = scan(text, rules?, context?: { filePath?, eventKind? });

// Replace matched spans with [REDACTED:CATEGORY] placeholders
const sanitized = redact(text, findings);

// Register a rule pack at startup
registerPack({ id: 'core-pii', rules: [...] });

// Inspect what's loaded
const rules = getLoadedRules();
```

## How `scan()` works

```
scan(text, rules, context)
  │
  ├─ scanText: for each rule that applies to context (appliesTo):
  │    ├─ MATCHERS[rule.matcher.type]  → keyword | regex spans
  │    ├─ postValidators (entropy, luhn) → drop spans that fail
  │    └─ requiresNearby               → drop / boost by nearby context
  │    (run over the original text AND a Unicode format-character-normalized copy)
  │
  ├─ scanEncoded: base64 runs and hex dumps decoded, scanned with the
  │    secret rules only, findings mapped back onto the encoded span
  │
  └─ returns MatchResult[] with:
       ruleId, category, severity, span {start, end}, rawMatch, confidence, variableName?
```

Matchers return zero or more `{start, end}` spans within the text. The engine wraps each span into a `MatchResult` with the rule's metadata attached. Matcher dispatch is a `MATCHERS` table keyed on the schema's matcher type, so a type the schema accepts but the engine doesn't handle is a compile error rather than a rule that silently matches nothing.

**Encoded secrets.** `base64 < .env` or `xxd -p .env` hands a rule a stream in which no secret appears literally. The engine finds bounded base64 and hex-dump runs (`xxd`, `hexdump -C`, `od -x`, …), decodes them as UTF-8, and scans the decoded text with the `secret`-category rules, so a redaction masks the encoding itself. Search and decode sizes are capped; a segment past the cap is refused whole and reported as unscanned rather than scanned as a truncated prefix.

## Matcher types

### keyword

Checks for literal keyword presence. Case-insensitive by default.

```json
{
  "type": "keyword",
  "keywords": ["password", "secret", "api_key", "private_key"],
  "caseSensitive": false
}
```

### regex

Applies a regular expression with optional flags.

```json
{
  "type": "regex",
  "pattern": "\\b(AKIA|ASIA)[A-Z0-9]{16}\\b",
  "flags": "g"
}
```

!!! note "Statement-level patterns"
    Use `\b` word boundaries in regex patterns to avoid false positives from substrings.

There is no third matcher type. Post-match checks are `postValidators` on a keyword or regex rule — see [Validators](#validators) below.

## How `redact()` works

Findings can overlap (two rules firing on the same value), and splicing overlapping spans independently would corrupt the output. So `redact()` first folds them into disjoint regions — sorted ascending, extending the open region while spans overlap — and the **highest-severity** finding in a region names its placeholder. Regions are then replaced back-to-front so earlier indices stay valid. Adjacent but disjoint spans stay separate.

The placeholder, `[REDACTED:<CATEGORY>]`, encodes the category (`PII`, `SECRET`, `FINANCIAL`, etc.) so downstream systems know what was removed.

## MatchResult shape

```typescript
interface MatchResult {
  ruleId: string; // e.g. "secrets/aws-access-key"
  category: DetectionCategory; // "pii" | "financial" | "secret" | "phi" | "code_context" | "code_flaw" | "custom" | "config"
  severity: Severity; // "critical" | "high" | "medium" | "low"
  span: { start: number; end: number };
  rawMatch: string; // the actual matched text
  confidence: number; // 0.0–1.0 (regex matches default to 0.9)
  variableName?: string; // for `NAME=value` / `name: value`, the name — never the value
}
```

## Loading rule packs

Rule packs are loaded at startup. Every pack under `rules/` is compiled into `packages/plugin-sdk/src/bundled-packs.generated.ts` at build time, and the plugin runtime scans with your **installed** snapshot of those packs (enabled packs only, from the local `installed_packs` table). On a machine attached to an organization's deployment, a pulled policy bundle that carries packs replaces them. In standalone tests (like the fixture-driven Vitest suite), packs are loaded directly from the `rules/` directory.

```typescript
// From the detection engine test setup:
const rule = JSON.parse(fs.readFileSync('rules/core-pii/email.json'));
const findings = scan(text, [Rule.parse(rule)]);
```

## Fixture-driven testing

Every rule **must** have a `fixtures/` directory with positive and negative test cases. The Vitest test at `packages/detections/test/engine.test.ts` auto-discovers all rule packs and runs their fixtures.

A fixture file is a JSON array:

```json
[
  { "label": "plain email", "text": "contact user@example.com", "shouldMatch": true },
  { "label": "no TLD", "text": "not-an-email@localhost", "shouldMatch": false }
]
```

CI fails if any fixture assertion fails. See [Writing Rules](../rules/writing-rules.md) for the full format.

## Adding a new matcher type

1. Create `packages/detections/src/matchers/<type>.ts` implementing `match(text, rule): Span[]`
2. Add the `type` literal to the `MatcherType` enum and the `Matcher` discriminated union in `packages/schema/src/zod/rule.ts`
3. Add its entry to the `MATCHERS` table in `engine.ts` (the table's type makes a missing entry a compile error)
4. Write fixtures

## Validators

A rule's `postValidators` run on each match, and a match that fails any of them is dropped. The names are a closed enum — anything else fails the rule's parse:

| Validator | File                    | What it checks                                                                    |
| --------- | ----------------------- | --------------------------------------------------------------------------------- |
| `luhn`    | `validators/luhn.ts`    | Credit/debit card number checksum                                                 |
| `entropy` | `validators/entropy.ts` | Shannon entropy of the match (defaults: `minLength` 20, `threshold` 3.5; overridable per rule) |

## ReDoS protection

Every bundled regex is timed in CI. At runtime, a pulled or custom regex rule that blows its timing budget — or has to be stopped mid-scan — is **quarantined**: cached as unsafe and dropped from later scans (`packages/plugin-sdk/src/rule-quarantine.ts`). Because that is a wall-clock judgement a loaded machine can reach about a rule that is fine, `aka detections unquarantine` clears the cached verdicts.
