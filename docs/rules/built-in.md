# Built-in Rules

AKA ships six rule packs out of the box with over 60 detection rules. All rules include positive and negative fixtures that run as part of the test suite.

## core-pii

**Location:** `rules/core-pii/`

Detects personally identifiable information commonly found in prompts. 13 rules.

| Rule ID              | Severity | Matcher | Description                                               |
| -------------------- | -------- | ------- | --------------------------------------------------------- |
| `email`              | medium   | regex   | Standard email addresses (`user@example.com`)             |
| `ssn`                | high     | regex   | US Social Security Numbers (`123-45-6789`)                |
| `phone-us`           | high     | regex   | US phone numbers incl. +1 and area codes                  |
| `phone-intl`         | high     | regex   | International E.164 numbers                               |
| `ip-address`         | low      | regex   | IPv4 addresses                                            |
| `dob`                | high     | regex   | Dates of birth (1900–2019)                                |
| `drivers-license-us` | high     | regex   | US driver's license numbers (requires license/DL context) |
| `passport-us`        | high     | regex   | US passport numbers (`A12345678`)                         |
| `home-address`       | medium   | keyword | Contextual address phrases                                |
| `name`               | medium   | keyword | Contextual name introduction phrases                      |
| `vin`                | medium   | regex   | Vehicle Identification Numbers (17 chars)                 |
| `mac-address`        | low      | regex   | MAC addresses (`AA:BB:CC:DD:EE:FF`)                       |
| `national-id`        | high     | regex   | Multi-country: Aadhaar, UK NI, Canadian SIN               |

## core-financial

**Location:** `rules/core-financial/`

Detects financial account information. 8 rules.

| Rule ID          | Severity | Matcher | Post-Validators | Description                                                         |
| ---------------- | -------- | ------- | --------------- | ------------------------------------------------------------------- |
| `credit-card`    | critical | regex   | `luhn`          | Visa, MC, Amex, Discover card numbers                               |
| `iban`           | high     | regex   | —               | International Bank Account Numbers                                  |
| `routing-number` | high     | regex   | —               | US ABA routing numbers (9 digits, requires routing/ABA/RTN context) |
| `swift`          | high     | regex   | —               | SWIFT/BIC codes (8 or 11 chars, requires SWIFT/BIC context)         |
| `cvv`            | high     | regex   | —               | Card security codes (contextual)                                    |
| `cusip`          | medium   | regex   | —               | US securities identifiers (requires CUSIP context)                  |
| `paypal`         | medium   | regex   | —               | PayPal transaction IDs (requires PayPal context)                    |
| `salary`         | low      | keyword | —               | Salary/compensation contextual phrases                              |

## secrets

**Location:** `rules/secrets/`

Detects credentials and tokens that should never be shared with AI models. 21 rules.

| Rule ID                   | Severity | Matcher | Post-Validators | Description                                                       |
| ------------------------- | -------- | ------- | --------------- | ----------------------------------------------------------------- |
| `aws-access-key`          | critical | regex   | `entropy`       | AWS Access Key IDs (AKIA/ASIA/AROA/etc.)                          |
| `aws-secret-key`          | critical | regex   | `entropy`       | AWS Secret Access Keys (40-char, requires aws-secret-key context) |
| `gcp-service-account`     | critical | regex   | —               | GCP service account emails                                        |
| `azure-connection-string` | critical | regex   | —               | Azure Storage/SQL connection strings                              |
| `openai-api-key`          | critical | regex   | `entropy`       | OpenAI API keys (`sk-proj-...`, `sk-...`)                         |
| `anthropic-api-key`       | critical | regex   | `entropy`       | Anthropic API keys (`sk-ant-...`)                                 |
| `stripe-live-key`         | critical | regex   | `entropy`       | Stripe live keys (`sk_live_`, `pk_live_`)                         |
| `slack-token`             | critical | regex   | —               | Slack tokens (`xoxb-`, `xoxp-`, `xapp-`)                          |
| `discord-token`           | critical | regex   | `entropy`       | Discord bot tokens (3-part format)                                |
| `sendgrid-key`            | critical | regex   | `entropy`       | SendGrid API keys (`SG.`)                                         |
| `twilio-key`              | critical | regex   | —               | Twilio Account SIDs (`AC...`)                                     |
| `gitlab-token`            | critical | regex   | `entropy`       | GitLab PATs (`glpat-`)                                            |
| `digitalocean-token`      | critical | regex   | —               | DigitalOcean PATs (`dop_v1_`)                                     |
| `datadog-key`             | critical | regex   | `entropy`       | Datadog API/APP keys                                              |
| `npm-token`               | critical | regex   | `entropy`       | npm access tokens (`npm_`)                                        |
| `heroku-api-key`          | critical | regex   | —               | Heroku API keys (UUID, requires heroku context)                   |
| `cloudflare-api-key`      | critical | regex   | `entropy`       | Cloudflare API tokens (40-char, requires cloudflare context)      |
| `pulumi-access-token`     | critical | regex   | `entropy`       | Pulumi access tokens (`pul-`)                                     |
| `terraform-cloud-token`   | critical | regex   | `entropy`       | Terraform Cloud user tokens                                       |
| `vault-token`             | critical | regex   | `entropy`       | HashiCorp Vault tokens (`hvs.`, `s.`)                             |
| `github-pat`              | critical | regex   | —               | GitHub PATs/OAuth/App tokens                                      |

## secrets-infra

**Location:** `rules/secrets-infra/`

Detects infrastructure-level secrets and credentials. 10 rules.

| Rule ID                | Severity | Matcher | Post-Validators | Description                                             |
| ---------------------- | -------- | ------- | --------------- | ------------------------------------------------------- |
| `ssh-private-key`      | critical | regex   | —               | RSA/DSA/EC/OpenSSH private key blocks                   |
| `db-connection-string` | critical | regex   | —               | DB URLs with credentials (`postgresql://user:pass@...`) |
| `jwt-token`            | high     | regex   | `entropy`       | JSON Web Tokens (3-part base64url)                      |
| `basic-auth-url`       | critical | regex   | —               | HTTP URLs with embedded credentials                     |
| `pgp-private-key`      | critical | regex   | —               | PGP private key blocks                                  |
| `bearer-token`         | high     | regex   | `entropy`       | Generic Authorization Bearer tokens                     |
| `env-key-value`        | high     | keyword | —               | Sensitive env vars with values                          |
| `password-field`       | high     | keyword | —               | JSON/YAML password/secret/token fields                  |
| `docker-config-auth`   | critical | regex   | —               | Docker `config.json` base64 auth                        |
| `kubeconfig-token`     | critical | regex   | —               | Kubernetes kubeconfig embedded tokens                   |

## core-phi

**Location:** `rules/core-phi/`

Detects protected health information (HIPAA-relevant). 6 rules.

| Rule ID            | Severity | Matcher | Description                                                |
| ------------------ | -------- | ------- | ---------------------------------------------------------- |
| `mrn`              | high     | regex   | Medical Record Numbers (contextual)                        |
| `hipaa-identifier` | high     | regex   | Health plan beneficiary / claim numbers                    |
| `icd10-code`       | medium   | regex   | ICD-10 diagnosis codes (requires ICD/diagnosis/dx context) |
| `ndc-code`         | medium   | regex   | National Drug Codes                                        |
| `genetic-data`     | high     | keyword | DNA/genetic data references                                |
| `biometric-ref`    | high     | keyword | Biometric data references                                  |

## core-code-context

**Location:** `rules/core-code-context/`

Detects internal code and infrastructure context that should not leave the organization. 8 rules.

| Rule ID           | Severity | Matcher | Description                                         |
| ----------------- | -------- | ------- | --------------------------------------------------- |
| `internal-ip`     | low      | regex   | RFC 1918 private IPs (10.x, 172.16-31.x, 192.168.x) |
| `localhost-ref`   | low      | regex   | localhost, 127.0.0.1, ::1 references                |
| `internal-domain` | low      | regex   | `.internal`, `.corp`, `.local` domains              |
| `file-path`       | low      | regex   | Absolute Unix (known roots) / Windows file paths    |
| `stack-trace`     | low      | regex   | Stack traces revealing internal structure           |
| `internal-url`    | low      | keyword | Dev/staging environment URLs                        |
| `feature-flag`    | low      | keyword | Feature flag / toggle names                         |
| `db-table-name`   | low      | keyword | Internal SQL table names                            |

## Running the test suite

```bash
pnpm --filter @alsoknownassecurity/detections test

# Watch mode during rule development
pnpm --filter @alsoknownassecurity/detections test -- --watch
```

Fixture tests auto-discover across all packs - no need to register new rules manually.

## Adding your own rules

See [Writing Rules](writing-rules.md) for the full guide. Create a new directory under `rules/`, add a `manifest.json`, your rule JSON files, and `fixtures/` — the test runner and pack loader auto-discover everything.
