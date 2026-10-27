---
comments: true
---

# Browser Extension

The browser extension (`plugins/browser-extension`) covers the surface the CLI plugins can't
reach: **web chat**. It intercepts submissions in ChatGPT and Claude.ai, scans them against the
same rules, and records findings to the same local store — `~/.aka/data/aka.db` — so browser
detections sit alongside Claude Code, Codex, and Antigravity ones in `aka findings` and the
dashboard.

It is a Chrome MV3 extension with a single permission (`nativeMessaging`), a toolbar popup, and
three content scripts (`bridge.js` and a page-world `tap.js` at `document_start`, `content.js` at
`document_idle`) matching:

- `https://chatgpt.com/*`
- `https://chat.openai.com/*`
- `https://claude.ai/*`

## How it works

The extension never talks to a network service. Its only privileged channel is a
**native-messaging host** running locally under Node:

1. A content script watches the composer's submit gesture on a supported provider and calls
   `preventDefault()` before the message leaves the page.
2. It relays the text to the MV3 service worker, which owns the one
   `chrome.runtime.connectNative` port (content scripts cannot open one themselves).
3. The native host scans the text with the same detection engine and returns a decision —
   **block**, **warn**, or **redact** — which the content script renders as an in-page banner.
4. An approved send is re-triggered programmatically, guarded so the re-fired gesture doesn't
   re-enter the interceptor.

Fail-open holds here too: a host that isn't installed, has died, or is wedged is bounded by a
3-second relay deadline, after which the send proceeds rather than the message silently vanishing.
The one exception is a message too large to scan within the time limit: it is **not** sent, and
a banner says so while your text stays in the composer.

A block banner names the `aka exception approve` command, so a user who accepts the risk can
grant an exception from a terminal — the page itself can't grant one.

### Web-chat capture

Enforcement (block, warn, redact) runs on every machine regardless of consent. *Recording* the
conversation — each reply with its tool calls and token usage — is a separate grant: turn on
**web-chat capture** under **Settings** in `aka dashboard`. The popup also names any site whose messages are not being
checked.

The popup names the claude.ai account a tab is signed in to. `aka extension account --on`
records each account it sees (its organization id and when it was seen, nothing from the chat)
so `aka enroll --list-detected` can list them; `aka extension account --off` stops that and
deletes the record. Both matter on a machine attached as a personal device — see the
[CLI README](https://github.com/akasecurity/ai-tc/blob/main/cli/README.md#the-browser-extension-chatgpt--claudeai-web-chat)
for exactly what is and isn't recorded there.

## Installing

Two halves — the native host (installed by the `aka` CLI) and the extension itself (loaded in
Chrome).

```bash
aka extension install
```

That writes the native-messaging host manifest and a launcher into Chrome's per-OS manifest
directory (plus the `HKCU\Software\Google\Chrome\NativeMessagingHosts\…` registry key on Windows),
pointing at a bundled Node host script. Then load the extension:

1. Open `chrome://extensions`.
2. Enable **Developer mode** (top right).
3. Click **Load unpacked** and select the extension's built `dist` directory — the path
   `aka extension install` prints when it finishes.

Check the host side any time with:

```bash
aka extension status
```

!!! note "Not yet on the Chrome Web Store"
    The extension package is unpublished (`private` in its `package.json`), so **Load unpacked**
    is the install path today. The extension id is pinned by the public `key` committed in
    `manifest.json`, so it stays stable across machines and rebuilds — which is what lets the
    native host's `allowed_origins` be locked to it.

Building from a workspace checkout:

```bash
pnpm --filter @akasecurity/plugin-browser-extension build
```

The standalone `aka` binary (Homebrew, Scoop, installer script) runs the native host itself; the
npm package runs it under the same Node.js runtime as the CLI. The registration points at the
link your package manager keeps current, so upgrades keep working — but if you moved or replaced
a hand-placed `aka`, run `aka extension install` again. `aka extension status` reports a
registration whose launcher is gone or points at a version other than the current one.

## Uninstalling

Remove the extension from `chrome://extensions`, and delete the native-messaging manifest and
launcher from the directory `aka extension status` reports (on Windows, also remove the registry
key above). Neither step touches `~/.aka` — see [Uninstalling](uninstalling.md).
