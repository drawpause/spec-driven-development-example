# Data Retention and Deletion

**Last Updated:** 2024-11-01

---

## Principles

- Retain only what is necessary for product functionality and legal compliance.
- Deleted data must be unrecoverable after the purge window closes.
- Honour GDPR right-to-erasure requests within 30 days.
- Audit logs are not subject to user-initiated deletion.

---

## Retention Schedule

| Data | Retention | Notes |
|---|---|---|
| User accounts | Until deletion request + 30-day hold | Hold allows self-service account recovery |
| Tasks (soft-deleted) | 90 days from `deleted_at` | Permanently purged after 90 days |
| File attachments | Until the parent task is purged | Deleted from S3 in the same purge job |
| Notifications | 1 year from `created_at` | Older records archived then purged |
| Refresh tokens (revoked) | 90 days from `revoked_at` | Kept for audit; not usable after revocation |
| Stripe webhook log | 1 year | Kept for billing reconciliation |
| Audit logs | 3 years | Cannot be deleted by users or admins |

---

## Account Deletion

When a user initiates account deletion from Settings:

1. Account is marked `deleted_at = now()`. All active refresh tokens are revoked.
2. The user is immediately logged out.
3. A confirmation email is sent: "Your account deletion is scheduled for {date}."
4. After the 30-day hold: all user records are purged — `users` row, `notifications`, comments, and avatars from S3.
5. Tasks created by the user are reassigned to the workspace owner; they are not deleted.
6. A final email is sent confirming permanent deletion. (Email is retained for 7 days post-send, then purged.)

---

## Workspace Deletion

When a workspace owner deletes a workspace:

1. All workspace members are notified by email.
2. Workspace is marked `deleted_at = now()`. The billing subscription is cancelled in Stripe immediately.
3. After the 30-day hold: workspace, all projects, tasks, comments, file attachments, and member records are permanently purged.

---

## GDPR Compliance

### Right to Access
Users can export all their data via `GET /account/export`. This produces a JSON archive containing their profile, tasks, comments, and notification history.

### Right to Erasure
Same as the account deletion flow above. Response is guaranteed within 30 days of the request.

### Record Keeping
All export and erasure requests are recorded in the audit log and are themselves not deletable.
