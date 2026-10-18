# pi-imessage

A [pi](https://pi.dev) extension that bridges the coding agent and Apple
iMessage on a Mac that is logged into Messages.app.

- **Inbound** — polls `~/Library/Messages/chat.db` (read-only connection, safe
  while Messages.app is running; 1 s interval, **off by default** — start with
  `/imessage poll on`). New allowlisted 1:1 DMs and self-chat are injected as
  user messages; own sent messages and their self-chat echoes are swallowed
  via fingerprints so replies from phone/devices still come through.
  A startup watermark seeds at the newest row, so old messages are never
  re-delivered.
- **Outbound** — the `imessage_send` tool (AppleScript → Messages.app).
  Recipients are allowlist-restricted; own identities are always allowed.
  Supports `group:` recipients and `threadId` threading; `deliveryMode:
  "attn"` sends silently.
- **@task** — inbound `@task <text>` (self or allowlisted) routes to the
  shared todo store for the headless 15-minute-cadence worker of the
  `global-todo` skill — never to the LLM — and drains the outbox for
  iMessage notifications.
- Denied (non-allowlisted) inbounds are logged to `denied.log` (JSONL) beside
  the config file; `/imessage status` shows the last denial.
- Group chats, SMS, attachments, and history search are intentionally
  unsupported in v1.

## Config

`~/.pi/agent/pi-imessage/config.json` (override with `PI_IMESSAGE_CONFIG`),
see [`config.example.json`](config.example.json). Keys: `meHandles` (own
identities), `allowFrom` (who may trigger the agent), `pollingEnabled`,
`pollIntervalMs`.

## Commands

```
/imessage [status | poll on|off | allow <h> | unallow <h> | me [h …] | send <to> <text>]
```

## Install

```sh
ln -s ~/code/pi-imessage ~/.pi/agent/extensions/pi-imessage
cd ~/code/pi-imessage && npm install   # builds better-sqlite3
# create ~/.pi/agent/pi-imessage/config.json (see config.example.json)
# restart pi, then: /imessage poll on
```

## Tests

```sh
node --test test/*.test.mjs   # unit + factory tests against temp fixture DBs
npm run typecheck
```

`test/real.test.mjs` is a manual smoke test against the real chat.db
(read-only); it skips cleanly when no Messages DB exists.
