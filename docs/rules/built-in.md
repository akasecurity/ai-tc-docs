# Built-in Rules

AKA ships two rule packs out of the box. Both include positive and negative fixtures that run as part of the test suite.

## core-pii

**Location:** `rules/core-pii/`

Detects personally identifiable information commonly found in prompts.

### email

| Field    | Value            |
| -------- | ---------------- |
| ID       | `core-pii/email` |
| Category | `pii`            |
| Severity | `medium`         |
| Matcher  | regex            |

**Pattern:**

```
\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b
```

Matches standard email addresses. Requires a TLD of at least 2 characters, so `user@localhost` does **not** match.

**Fixtures:**

| Text                                    | Match? |
| --------------------------------------- | ------ |
| `Contact me at user@example.com please` | ✅     |
| `user@localhost`                        | ❌     |
| `not-an-email`                          | ❌     |

---

### ssn

| Field    | Value          |
| -------- | -------------- |
| ID       | `core-pii/ssn` |
| Category | `pii`          |
| Severity | `high`         |
| Matcher  | regex          |

**Pattern:**

```
\b(?!000|666|9\d{2})\d{3}-(?!00)\d{2}-(?!0000)\d{4}\b
```

Matches US Social Security Numbers in `###-##-####` format. The pattern excludes known invalid prefixes (000, 666, 900–999) and all-zero groups.

**Fixtures:**

| Text                           | Match? |
| ------------------------------ | ------ |
| `SSN: 123-45-6789`             | ✅     |
| `000-12-3456` (invalid prefix) | ❌     |
| `123456789` (no dashes)        | ❌     |

---

## secrets

**Location:** `rules/secrets/`

Detects credentials and tokens that should never be shared with AI models.

### aws-access-key

| Field           | Value                    |
| --------------- | ------------------------ |
| ID              | `secrets/aws-access-key` |
| Category        | `secret`                 |
| Severity        | `critical`               |
| Matcher         | regex                    |
| Post-validators | `entropy`                |

**Pattern:**

```
\b(AKIA|ASIA|AROA|AIDA|ANPA|ANVA|APKA)[A-Z0-9]{16}\b
```

Matches AWS access key IDs. All valid AWS key prefixes are covered (AKIA = long-term, ASIA = STS temporary, AROA = assumed role, etc.). Total length is always 20 characters.

**Fixtures:**

| Text                                  | Match? |
| ------------------------------------- | ------ |
| `const key = "AKIAIOSFODNN7EXAMPLE";` | ✅     |
| `token: ASIAIOSFODNN7EXAMPLE`         | ✅     |
| `AKIASHORT` (too short)               | ❌     |

!!! warning "Secret exposure is critical"
AWS access keys grant programmatic access to your AWS account. A leaked AKIA key with no restrictions can be used to provision resources, exfiltrate S3 data, or rack up billing charges.

---

### github-pat

| Field    | Value                |
| -------- | -------------------- |
| ID       | `secrets/github-pat` |
| Category | `secret`             |
| Severity | `critical`           |
| Matcher  | regex                |

**Pattern:**

```
\bghp_[A-Za-z0-9]{36}\b|\bgho_[A-Za-z0-9]{36}\b|\bghs_[A-Za-z0-9]{36}\b
```

Matches GitHub Personal Access Tokens, OAuth tokens, and GitHub Apps tokens. All modern GitHub tokens follow a `<prefix>_<36 alphanumeric chars>` format.

| Prefix | Type                                            |
| ------ | ----------------------------------------------- |
| `ghp_` | Personal access token (classic or fine-grained) |
| `gho_` | OAuth token                                     |
| `ghs_` | GitHub Apps installation token                  |

**Fixtures:**

| Text                                                | Match? |
| --------------------------------------------------- | ------ |
| `token: ghp_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890`   | ✅     |
| `GH_TOKEN=ghs_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890` | ✅     |
| `ghp_short`                                         | ❌     |
| bare 40-char string without prefix                  | ❌     |

---

## Running the test suite

```bash
# Run all fixture tests
pnpm --filter @aka/detections test

# Watch mode during rule development
pnpm --filter @aka/detections test -- --watch
```

Expected output:

```
 ✓ core-pii/email (2 tests)
 ✓ core-pii/ssn (2 tests)
 ✓ secrets/aws-access-key (4 tests)
 ✓ secrets/github-pat (4 tests)
 ✓ redact (2 tests)

 Tests  14 passed (14)
```

## Adding your own rules

See [Writing Rules](writing-rules.md) for the full guide. To add a rule to an existing pack, add the JSON file and its fixture — the test runner auto-discovers everything in `rules/`.
