# UX Flows

**Last Updated:** 2024-11-01

---

## Onboarding Flow

### New User (Direct Signup)

```
Sign Up Page
    |
    v
Email Verification Pending
  (user receives verification email)
    |
    | (clicks link in email)
    v
Create Workspace
  (enter workspace name; slug auto-generated, editable)
    |
    v
Invite Teammates
  (enter emails; skippable)
    |
    v
Create First Project
  (enter name; description optional)
    |
    v
Empty Project View
  (inline prompt: "Add your first task")
```

**Edge cases:**
- Verification link expired (> 24h): show "Link expired" page with a "Resend verification email" button.
- Workspace slug already taken: show inline error with up to 3 auto-generated alternatives.
- User closes the tab mid-onboarding: on next login, resume at the earliest incomplete onboarding step.

---

## Task Lifecycle

Tasks move through the following states. All transitions are explicit (user action), not automatic.

```
        +----------+
        |   todo   |<-----------------------------+
        +----+-----+                              |
             |                                   |
             v                                   |
      +-------------+                            |
      | in_progress |                            |
      +------+------+                            |
             |                                   |
             v                                   |
       +-----------+       +-----------+         |
       | in_review +-----> |   done    +---------+
       +-----------+       +-----+-----+  (re-open)
             |                   |
             |       +-----------+
             +-----> | cancelled |
                     +-----------+
```

Rules:
- Any status can transition to `cancelled`.
- `done` and `cancelled` are considered closed; they can be re-opened by moving back to `todo`.
- Every status change is logged as a task activity entry (visible in the task detail view).

---

## Notification Inbox

The notification inbox is accessible via the bell icon in the top navigation.

**Components:**
1. **Unread count badge** — shown on the bell icon. Updated in real time via WebSocket. Hidden when count is 0.
2. **Inbox panel** — slides in from the right. Shows notifications sorted newest-first, grouped by day ("Today", "Yesterday", "Earlier").
3. **Mark all read** — button at the top right of the panel. Sets `read_at` on all unread records.
4. Each notification row is clickable and navigates to the relevant task.

**Empty state (no notifications):**
- Heading: "You're all caught up"
- No body text, no CTA. Clean.

---

## Notification Preferences Flow

```
Top Nav > User Avatar > Settings > Notifications tab

+----------------------------------------------+
| Notification Preferences                     |
+----------------------------------------------+
| Event                    In-App   Email      |
| Task assigned to you       (*)     (*)       |
| @mentioned in a comment    (*)     (*)       |
| Task you own completed     (*)     ( )       |
| Due date approaching       (*)     (*)       |
| Added to a project         ( )     ( )       |
+----------------------------------------------+
  (*) = enabled   ( ) = disabled
```

Toggles auto-save on change (no explicit save button). A toast appears at the bottom of the screen: "Preferences saved."

---

## Billing Upgrade Flow

```
Workspace Settings > Billing

Current Plan: Free (3/3 projects used)
  [Upgrade to Pro]
        |
        v
Upgrade Modal
  Select seats: [--] 5 [++]  (min: current member count)
  Plan: Pro @ $12/seat/mo
  Total: $60/mo
  [Add payment method] | [Cancel]
        |
        v (Stripe Elements embedded)
  Enter card details
  [Subscribe] | [Cancel]
        |
        v
Success screen: "You're on Pro."
  Confetti animation (one-time).
  [Back to workspace]
```

If payment fails during checkout: show Stripe's inline error message. Do not close the modal.
