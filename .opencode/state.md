# Current State

## Objective
Custom pi extension (TypeScript, pi v0.84.2) reimplementing removed `@treentity/pi-imessage`
with simplifications: inbound poll of chat.db by ROWID>watermark (watermark = MAX at boot,
no replay); outbound via osascript→Messages.app; NO history search, NO attachments,
NO multi-terminal, NO group chats; self-chat detection from stored phone/email;
polling disabled by default (command to enable); allowlist-gated inbound AND outbound.

## Acceptance Criteria
- [x] tsc clean
- [x] unit tests: fixture DB (test/db.test.mjs) — all pass
- [x] real-DB sanity (test/real.test.mjs) — all pass on real chat.db (read-only)
- [x] factory tests: mock ExtensionAPI (test/factory.test.mjs) — all pass
- [x] E2E: `pi -p "/imessage status"` loads extension, watermark 101076, self learned,
      status block correct, no accidental sends
- [ ] User runs TUI: /imessage poll on, allow a peer, live message flow (manual)

## Current Phase
DONE — implementation + verification complete. Only manual TUI live-test remains.

## Completed
- src/config.ts — load/save ~/.pi/agent/pi-imessage/config.json (PI_IMESSAGE_CONFIG),
  self handles, allowlist, pollingEnabled=false, pollIntervalMs
- src/db.ts — read-only/WAL better-sqlite3 (query_only=1), PI_IMESSAGE_DB override,
  MAX(ROWID) watermark (no replay), learnSelf (P:/E: accounts, iMessage-prefixed or bare),
  pollNew (raw candidates, cmj rowid cursor), isGroupChat (handles-first w/ self,
  identifier P;+;P legacy, group: prefix), peerOf (handle-set-first; sender-fallback
  ONLY when handles.length<=1 && !isFromMe, using h.id address; identifier P:/E: last),
  chatHandles, handleValue (rowid→address)
- src/send.ts — osascript send (quoted), PI_IMESSAGE_SEND_CMD test hook
- src/index.ts — boot (watermark init, learnSelf merge, status note), polling
  setInterval+unref (off by default), inbound handleRow (echo / group / item types /
  service / empty / replace / system filters, allowlist gate, rememberPeer, deliver
  plain if isIdle else followUp), statusText, /imessage command (status|poll on|off|
  allow|unallow|me|send [peer]), imessage_send tool (allowlist-gated, last-peer default),
  session_shutdown stops timer
- test/ — fixtures.mjs (6-row cmj, wide message table, real-shaped handle),
  db.test.mjs, real.test.mjs, factory.test.mjs (mock pi API)
- E2E verified via real pi 0.84.2 one-shot (send hooked)
- README + skill from prior session

## Relevant Files
- src/index.ts (boot/poll/handleRow/status/command/tool), src/db.ts, src/config.ts, src/send.ts
- test/{fixtures,db.test,real.test,factory.test}.mjs
- package.json (pi manifest → src/index.ts), linked ~/.pi/agent/extensions/pi-imessage

## Decisions
- Filter in JS (handleRow), SQL returns raw candidates — cursor stays correct,
  matches original semantics (message_date > lastTimestamp + rowid tie-break)
- No history search, no attachments (deferred, per requirement)
- Outbound gated same as inbound (own handles always allowed)
- GLOB gotcha: % = exactly one char (never used for prefixes)
- better-sqlite3 ESM: default import is the constructor (named import fails)

## Tests
node test/db.test.mjs ✔ ; node test/real.test.mjs ✔ ; node test/factory.test.mjs ✔
npx tsc --noEmit ✔

## Failures
- (none open; previous failures all fixed: fixture handle.id shape, learnSelf
  account formats, group detection, last-peer, mock registerTool shape,
  send-gate vs allowlist in tests, note prefix, empty-allowlist text, invalid regex)

## Blockers
- (none)

## Next Action
Manual TUI: open pi, /imessage poll on + /imessage allow +PEER, send a real iMessage
from another device, confirm delivery + /imessage status; then /imessage send test.
Optional: E2E agent-driven send via hook (pi -p "send me a test iMessage").
