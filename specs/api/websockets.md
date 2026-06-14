# WebSocket API Specification

```yaml
spec: api/websockets
status: active
last_updated: 2026-06-14
endpoint: wss://api.relay.app/v1/ws
owns:
  - src/server/ws/**
depends_on:
  - features/auth
  - features/notifications
```

## Summary

One persistent WebSocket per browser tab delivers real-time task and notification events after a mandatory auth handshake and per-channel subscription.

## Invariants

- **INV-WS-1:** A client MUST send an `auth` message within 5s of connecting; otherwise the server MUST close with code `4001`.
- **INV-WS-2:** If the auth token is invalid or expired, the server MUST close with code `4003` and MUST NOT deliver any event.
- **INV-WS-3:** The server MUST verify the authenticated user has access to a channel before subscribing; on denial it MUST send `subscribe.denied` and MUST NOT close the connection.
- **INV-WS-4:** Every message in both directions MUST be a JSON object with a `type` field.
- **INV-WS-5:** A malformed or unrecognized client message MUST close the connection with code `1008`.
- **INV-WS-6:** The server MUST emit events only to subscribers of the matching channel; an event MUST NOT reach a non-subscribed or unauthorized client.
- **INV-WS-7:** `task.updated` payloads MUST include only changed fields plus `id` and `updated_at`.

## Contract

### Handshake
```json
// client → server, within 5s
{ "type": "auth", "token": "eyJ..." }
// server → client
{ "type": "auth.ok", "user_id": "usr_01" }
```

### Subscribe
```json
{ "type": "subscribe", "channel": "project:prj_01" }
{ "type": "subscribed", "channel": "project:prj_01" }
// or, on denial:
{ "type": "subscribe.denied", "channel": "project:prj_01" }
```

| Channel pattern | Example | Delivers |
|---|---|---|
| `project:{id}` | `project:prj_01` | `task.created`, `task.updated`, `task.deleted` |
| `notifications:{user_id}` | `notifications:usr_01` | `notification.new` |

### Server events
```json
{ "type": "task.created",  "channel": "project:prj_01", "data": { "id":"tsk_05", "title":"…", "status":"todo", "assignee_id":"usr_02", "due_at":null, "created_by":"usr_01", "created_at":"…" } }
{ "type": "task.updated",  "channel": "project:prj_01", "data": { "id":"tsk_05", "changes": { "status":"in_progress", "updated_at":"…" } } }
{ "type": "task.deleted",  "channel": "project:prj_01", "data": { "id":"tsk_05" } }
{ "type": "notification.new", "channel": "notifications:usr_01", "data": { "id":"notif_09", "event":"task_assigned", "payload": {…}, "read_at":null, "created_at":"…" } }
```

### Close codes
| Code | Meaning |
|---|---|
| `1000` | Normal closure |
| `1008` | Policy violation — malformed/unrecognized message (INV-WS-5) |
| `4001` | Auth timeout — no `auth` within 5s (INV-WS-1) |
| `4003` | Auth failed — invalid/expired token (INV-WS-2) |

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-WS-1,2 | `src/server/ws/handshake.ts` | `authHandshake` |
| INV-WS-3,6 | `src/server/ws/channels.ts` | `subscribe`, `authorizeChannel`, `publish` |
| INV-WS-4,5 | `src/server/ws/protocol.ts` | `parseMessage` |
| INV-WS-7 | `src/server/ws/events.ts` | `taskUpdatedEvent` |

## Acceptance

- **AC-WS-1** (INV-WS-1): GIVEN a connection WHEN no `auth` within 5s THEN server closes with `4001`.
- **AC-WS-2** (INV-WS-2): GIVEN an expired token WHEN sent as `auth` THEN server closes with `4003` and emits no events.
- **AC-WS-3** (INV-WS-3): GIVEN a user without access WHEN subscribing to that `project:` channel THEN server replies `subscribe.denied` and keeps the connection open.
- **AC-WS-4** (INV-WS-6): GIVEN a task update in `prj_01` WHEN a client is subscribed only to `prj_02` THEN it receives nothing.
- **AC-WS-5** (INV-WS-5): GIVEN a non-JSON frame THEN server closes with `1008`.

## Verify

```bash
npm test -- ws                    # INV-WS-1..7
npm run test:int -- ws-authz      # AC-WS-3,4 (channel authorization isolation)
```
