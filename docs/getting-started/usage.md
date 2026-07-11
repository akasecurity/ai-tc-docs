# Usage

The most common way to work with AKA day-to-day is the `/aka:*` slash commands, run from inside a Claude Code session. They're all read-only and backed by the local SQLite store (`~/.aka/data/aka.db`), so they work with zero backend.

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

See the [Claude Code plugin guide](../plugin/claude-code.md#read-surface-slash-commands) for the full detail on each command.
