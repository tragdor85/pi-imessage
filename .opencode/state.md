# Current State

## Objective
Stop my own outbound iMessages (this pi session's sends) from coming back as
inbound, while still receiving self-chat/DM messages the user sends from
phone/devices and replying to the number that sent. — DONE & COMMITTED.

## How it works (verified on real chat.db)
Self-chat iMessage sent from THIS Mac via Messages.app creates two rows:
local sent row (is_from_me=1, filtered) + server ECHO row (is_from_me=0,
sender = own handle, fresh GUID — server re-IDs it, so no GUID join possible).
Echo is shape-identical to a user self-chat text typed on the phone, so filter
is send-side: recordOutboundEcho(text, peer) on send success; handleRow
swallows one matching inbound row (exact text + sameHandle peer, TTL 2 min,
ring of 50). One-shot consume → user re-typing my text still delivers.
Reply routing already correct: lastPeer set by last inbound → replies go to
the sending number (self-chat → own number, DM → peer).

## Completed
- src/index.ts: EchoEntry + ECHO_TTL_MS/ECHO_MAX, Bridge.outboundEchoes,
  recordOutboundEcho + isSelfEcho (one-shot), doSend success hook, handleRow
  suppression (note + persist), header doc.
- test/factory.test.mjs: new block — send to own self-chat, insert is_from_me=0
  echo row + user row + re-typed row → echo suppressed, both genuine rows
  delivered (in rowid order), suppression noted, reply routes to SELF_PHONE,
  watermark advanced. 3 total delivered.
- Verified: tsc ✔, db.test ✔, factory.test ✔, real.test ✔ (wm 101094),
  real-sweep ✔ (101,513 dates, 0 throws)

## Relevant Files
- src/index.ts (handleRow ~L300, doSend ~L354, record/isSelfEcho ~L242)
- test/factory.test.mjs (self-echo block before session_shutdown)
- .opencode/inspect-echo.mjs (UNTRACKED real-DB diagnostic; keep or delete)

## Decisions
- One-shot consumption (not window-suppress) — no permanent loss of a genuine
  re-typed identical message.
- In-memory session-scoped fingerprints (not persisted): pi restart between
  send and echo → at most one stray echo, self-heals.
- No GUID join: server assigns a new GUID to the echo (verified: AFBE4219
  sent → 53C9302E echo, joinrowid 101093→101094).

## Tests
node test/db.test.mjs ✔ | node test/real.test.mjs ✔ |
node test/real-sweep.mjs ✔ | node test/factory.test.mjs ✔ (3 delivered) |
npx tsc --noEmit ✔

## Failures
- (none)

## Blockers
- (none)

## Next Action
Manual live check (optional but recommended): restart pi extension, poll on,
agent sends a self-chat iMessage, confirm NO "[iMessage self-chat]" echo
arrives and a phone-typed self-chat message still does. Then delete or keep
.opencode/inspect-echo.mjs.

---

## History
v2 build (poll>watermark no replay; allowlist both directions; no
search/attachments/multi-terminal/groups) + date-scale auto-detect fix
(committed d89b3c8). E2E clean on pi 0.84.2.
