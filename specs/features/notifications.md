# Feature: Notifications

```yaml
spec: features/notifications
status: active
last_updated: 2026-06-14
owns:
  - src/notifications/**
  - src/workers/notify.ts
  - src/workers/send-email.ts
depends_on:
  - api/rest
  - api/websockets
  - data/schema
  - architecture
```

## Summary

Relay notifies users of relevant activity via in-app (WebSocket + DB) and email channels, delivered by background workers and configurable per event type and channel.

## Event Catalog

| Event key | In-app | Email | Default |
|---|---|---|---|
| `task_assigned` | yes | yes | on |
| `mention` | yes | yes | on |
| `task_completed` | yes | no | on |
| `due_reminder` | yes | yes | on |
| `member_added` | yes | no | off |

These keys are authoritative for `users.notification_preferences` and `notifications.event`.

## Invariants

- **INV-NOTIF-1:** Notification delivery MUST NOT block the triggering API request; the request enqueues a `notify` job and returns (see `architecture` INV-ARCH-2).
- **INV-NOTIF-2:** An in-app notification MUST be persisted and pushed within 5s of the triggering event.
- **INV-NOTIF-3:** Email MUST be sent only when the event's catalog `Email` column is `yes` AND the recipient's preference for that channel is enabled.
- **INV-NOTIF-4:** Failed `notify`/`send_email` jobs MUST retry up to 3 times with exponential backoff.
- **INV-NOTIF-5:** `notifications.event` MUST be one of the Event Catalog keys; no other value is writable.
- **INV-NOTIF-6:** Marking all read MUST set `read_at = now()` on every currently-unread row for that user and no others.
- **INV-NOTIF-7:** The unread count pushed over WebSocket MUST equal `COUNT(*) WHERE user_id = ? AND read_at IS NULL`.
- **INV-NOTIF-8:** A disconnected recipient MUST still see the notification persisted on next load (WS is delivery, DB is source of truth).

## Contract

Delivery pipeline:
```
API mutation completes
   └─ enqueue notify{recipient, event, payload}        (request returns here)

worker notify:
   1. insert notifications row
   2. push WS notification.new to recipient's channels  (api/websockets)
   3. if email enabled for (event, user): enqueue send_email

worker send_email:
   1. render template
   2. send via SES
```

Endpoints: `GET /notifications`, `POST /notifications/read-all` (see `api/rest`). WS event: `notification.new` (see `api/websockets`).

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-NOTIF-1 | `src/notifications/emit.ts` | `emitNotification` (enqueue only) |
| INV-NOTIF-2 | `src/workers/notify.ts` | `handleNotify` |
| INV-NOTIF-3 | `src/notifications/preferences.ts` | `shouldEmail` |
| INV-NOTIF-4 | `src/workers/queue.ts` | `RETRY_POLICY` |
| INV-NOTIF-5 | `src/notifications/events.ts` | `EVENT_KEYS` |
| INV-NOTIF-6 | `src/notifications/read.ts` | `markAllRead` |
| INV-NOTIF-7 | `src/notifications/count.ts` | `unreadCount` |

## Acceptance

- **AC-NOTIF-1** (INV-NOTIF-2): GIVEN a task is assigned WHEN ≤5s elapse THEN the assignee has a persisted `task_assigned` notification.
- **AC-NOTIF-2** (INV-NOTIF-3): GIVEN a user with email disabled for `task_assigned` WHEN assigned THEN no `send_email` job is enqueued.
- **AC-NOTIF-3** (INV-NOTIF-1): GIVEN an assignment request WHEN measured THEN the response returns before any email send begins.
- **AC-NOTIF-4** (INV-NOTIF-6): GIVEN unread notifications WHEN `read-all` is called THEN every previously-unread row has `read_at` set and unread count is 0.
- **AC-NOTIF-5** (INV-NOTIF-8): GIVEN a recipient with no WS connection WHEN an event fires THEN the notification appears in `GET /notifications` on next load.

## Verify

```bash
npm test -- notifications              # INV-NOTIF-3,5,6,7
npm run test:int -- notify-latency     # INV-NOTIF-1,2 (≤5s)
npm run test:queue -- retries          # INV-NOTIF-4
```
