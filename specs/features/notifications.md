# Feature: Notifications

**Last Updated:** 2024-11-01
**Status:** Active

---

## Summary

Relay notifies users of activity relevant to them via in-app notifications and email. Users can configure their preferences per event type and delivery channel.

---

## Goals
- Inform users of task assignments, mentions, and status changes in near-real-time.
- Avoid notification fatigue through sensible defaults and per-user preferences.
- Deliver notifications reliably via background jobs, never blocking the triggering request.

## Non-Goals
- Mobile push notifications (future).
- Slack or Teams integration (planned, not in v1).
- Digest emails or weekly summaries (future).
- Notification grouping or threading.

---

## Event Types

| Event | In-App | Email | Default |
|---|---|---|---|
| Task assigned to you | Yes | Yes | On |
| You are @mentioned in a comment | Yes | Yes | On |
| A task you own is marked done | Yes | No | On |
| Due date approaching (24h warning) | Yes | Yes | On |
| You are added to a project | Yes | No | Off |

---

## Requirements

### Functional
- [ ] Users receive in-app notifications for all enabled event types.
- [ ] Users receive email notifications for events where email is enabled and the user has it turned on.
- [ ] Users can toggle each event type on/off independently for in-app and email channels.
- [ ] Notifications can be marked as read individually or all at once.
- [ ] Unread notification count is shown in the UI nav and updated in real time via WebSocket.

### Non-Functional
- In-app notifications must appear within 5 seconds of the triggering event.
- Email delivery must not block the API request that triggered the event.
- The notification worker must retry failed jobs up to 3 times with exponential backoff.

---

## Delivery Flow

1. A mutation completes in the API (e.g., a task is assigned).
2. The API enqueues a `notify` job in Redis via BullMQ and returns its response to the caller.
3. The notification worker picks up the job and:
   a. Writes a `notifications` record to the DB.
   b. Pushes a `notification.new` WebSocket event to all of the recipient's connected clients.
   c. If email is enabled for this event type: enqueues a separate `send_email` job.
4. The email worker renders the template and sends via AWS SES.

---

## Acceptance Criteria

- [ ] Assigning a task creates an in-app notification for the assignee within 5 seconds.
- [ ] If email is enabled, the assignee receives an email within 60 seconds of assignment.
- [ ] Marking all notifications as read sets `read_at = now()` on every unread record for that user.
- [ ] Turning off email for "task assigned" stops further email delivery for that event immediately.
- [ ] A user who is not connected via WebSocket still sees the notification on their next page load.
