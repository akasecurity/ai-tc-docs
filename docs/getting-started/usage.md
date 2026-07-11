# Usage

The most common way to work with AKA day-to-day is the `/aka:*` slash commands, run from inside a Claude Code or Claude Desktop session — the same plugin, same commands either way. They're all read-only and backed by the local SQLite store (`~/.aka/data/aka.db`), so they work with zero backend.

| Command            | Shows                                                                           |
| ------------------- | -------------------------------------------------------------------------------- |
| `/aka:health`      | detection activity, action breakdown, category coverage, last 7 days           |
| `/aka:findings`    | recent findings with rule, category, severity, action, and the masked match    |
| `/aka:recommend`   | findings grouped by category (ranked by severity, then frequency) + next steps |
| `/aka:audit`       | the decision log — recent detections and the action AKA took                   |
| `/aka:scan`        | scan working-tree source files for insecure code patterns (OWASP Top 10)       |
| `/aka:tokens`      | token usage per provider/model (reconciled from transcripts) + estimated cost  |
| `/aka:exceptions`  | active detection exceptions — masked value, rule, scope, expiry, uses          |
| `/aka:detections`  | installed detection packs — version, rules, enabled, policy, update available  |

### `/aka:health`

Run this as a general pulse check — after onboarding to confirm AKA is actually catching things, or periodically to sanity-check posture without digging into individual findings.

### `/aka:findings`

Reach for this right after something gets flagged, to see exactly what matched and how it was handled — the natural next step after a block or warn message in your session.

### `/aka:recommend`

Run this when deciding what to tighten next. Most rules ship under the log-only Monitor policy, and this surfaces which categories have enough real activity to justify promoting to Warn, Redact, or Block.

### `/aka:audit`

Use this to answer "why did AKA do that?" after the fact — reconstructing the sequence of decisions for a session, or for a compliance/incident review.

### `/aka:scan`

Run this proactively rather than waiting for a hook to catch something live — before a commit or PR, after pulling in third-party code, or any time you suspect a secret slipped into the working tree.

### `/aka:tokens`

Check this when you want visibility into spend — tracking usage across providers/models or reconciling cost without leaving the session.

### `/aka:exceptions`

Check this before assuming a detection will fire — if something you expected to be blocked wasn't, an active exception is the first thing to rule out. It won't tell you how to grant one; that's intentionally only available out-of-band via the `aka` CLI.

### `/aka:detections`

Check this before writing a new rule (to avoid duplicating an existing pack) or before running `aka detections update`, to see what's currently installed versus what's available.

See the [Claude Code plugin guide](../plugin/claude-code.md#read-surface-slash-commands) for the full detail on each command.
