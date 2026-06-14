# WebSocket API Specification

**Last Updated:** 2024-11-01
**Endpoint:** `wss://api.relay.app/v1/ws`

---

## Overview

Relay maintains one persistent WebSocket connection per browser tab for real-time updates. The client authenticates immediately after connecting, then subscribes to channels for the resources it has open.

---

## Message Format

All messages — in both directions — are JSON objects with a `type` field.

```json
{ "type": "event_name", ...other fields }
```

---

## Connection Lifecycle

### 1. Connect
Client opens a WebSocket to `wss://api.relay.app/v1/ws`.

### 2. Authenticate
Client must send an `auth` message within 5 seconds of connecting, or the server closes the connection with code `4001`.

```json
// Client -> Server
{ "type": "auth", "token": "eyJ..." }

// Server -> Client (success)
{ "type": "auth.ok", "user_id": "usr_01" }
```

If the token is invalid or expired, the server closes with code `4003`.

### 3. Subscribe to channels
After auth, the client subscribes to the resources it wants real-time updates for.

```json
// Client -> Server
{ "type": "subscribe", "channel": "project:prj_01" }

// Server -> Client
{ "type": "subscribed", "channel": "project:prj_01" }
```

Available channel patterns:
| Pattern | Example | Receives |
|---|---|---|
| `project:{id}` | `project:prj_01` | Task created/updated/deleted in that project |
| `notifications:{user_id}` | `notifications:usr_01` | New notifications for that user |

The server validates that the authenticated user has access to the requested channel. If not, it sends a `subscribe.denied` message instead of closing.

---

## Server-Sent Events

### task.created
```json
{
  "type": "task.created",
  "channel": "project:prj_01",
  "data": {
    "id": "tsk_05",
    "title": "Review PR #42",
    "status": "todo",
    "assignee_id": "usr_02",
    "due_at": null,
    "created_by": "usr_01",
    "created_at": "2024-11-01T14:00:00Z"
  }
}
```

### task.updated
Includes only the fields that changed.

```json
{
  "type": "task.updated",
  "channel": "project:prj_01",
  "data": {
    "id": "tsk_05",
    "changes": { "status": "in_progress", "updated_at": "2024-11-01T15:30:00Z" }
  }
}
```

### task.deleted
```json
{
  "type": "task.deleted",
  "channel": "project:prj_01",
  "data": { "id": "tsk_05" }
}
```

### notification.new
```json
{
  "type": "notification.new",
  "channel": "notifications:usr_01",
  "data": {
    "id": "notif_09",
    "event": "task_assigned",
    "payload": { "task_id": "tsk_05", "task_title": "Review PR #42" },
    "read_at": null,
    "created_at": "2024-11-01T14:00:00Z"
  }
}
```

---

## Close Codes

| Code | Meaning |
|---|---|
| `1000` | Normal closure |
| `4001` | Auth timeout — no auth message received within 5 seconds |
| `4003` | Auth failed — invalid or expired token |
| `1008` | Policy violation — client sent an unrecognized or malformed message |
