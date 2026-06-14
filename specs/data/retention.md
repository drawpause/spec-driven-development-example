# Data Retention and Deletion

```yaml
spec: data/retention
status: active
last_updated: 2026-06-14
owns:
  - src/retention/**
  - src/workers/purge.ts
depends_on:
  - data/schema
  - ops/security
```

## Summary

How long Relay keeps each data class, how deletion and GDPR requests are honored, and the purge guarantees that make deleted data unrecoverable.

## Invariants

- **INV-RET-1:** Each data class MUST be purged on or before its retention deadline in the schedule below; nothing is kept past it except audit logs.
- **INV-RET-2:** After the purge window closes, deleted data MUST be unrecoverable (hard-deleted from Postgres and S3).
- **INV-RET-3:** GDPR right-to-erasure requests MUST complete within 30 days.
- **INV-RET-4:** Audit logs MUST NOT be deletable by users or admins.
- **INV-RET-5:** Account deletion MUST revoke all refresh tokens and log the user out immediately.
- **INV-RET-6:** On account purge, tasks created by the user MUST be reassigned to the workspace owner, not deleted.
- **INV-RET-7:** Every export and erasure request MUST be recorded in the (non-deletable) audit log.

## Contract

### Retention schedule
| Data | Retention | Notes |
|---|---|---|
| User accounts | deletion request + 30-day hold | hold enables self-service recovery |
| Tasks (soft-deleted) | 90 days from `deleted_at` | then purged |
| File attachments | until parent task purged | deleted from S3 in same job |
| Notifications | 1 year from `created_at` | archived then purged |
| Refresh tokens (revoked) | 90 days from `revoked_at` | audit; unusable after revoke |
| Stripe webhook log | 1 year | billing reconciliation |
| Audit logs | 3 years | non-deletable (INV-RET-4) |

### Account deletion flow
1. `users.deleted_at = now()`; revoke all refresh tokens; log out (INV-RET-5).
2. Send "deletion scheduled for {date}" email.
3. After 30-day hold: purge `users` row, `notifications`, comments, avatars from S3 (INV-RET-2).
4. Reassign the user's tasks to workspace owner (INV-RET-6).
5. Send final confirmation email (retained 7 days post-send, then purged).

### Workspace deletion flow
1. Email all members.
2. `workspaces.deleted_at = now()`; cancel Stripe subscription immediately.
3. After 30-day hold: purge workspace, projects, tasks, comments, attachments, memberships.

### GDPR
- Right to access: `GET /account/export` → JSON archive (profile, tasks, comments, notifications).
- Right to erasure: account deletion flow; guaranteed ≤ 30 days (INV-RET-3).

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-RET-1,2 | `src/workers/purge.ts` | `runPurgeCycle`, `RETENTION_SCHEDULE` |
| INV-RET-3 | `src/retention/erasure.ts` | `scheduleErasure` |
| INV-RET-5 | `src/retention/account.ts` | `deleteAccount` |
| INV-RET-6 | `src/retention/account.ts` | `reassignTasksToOwner` |
| INV-RET-7 | `src/retention/audit.ts` | `recordRequest` |

## Acceptance

- **AC-RET-1** (INV-RET-1): GIVEN a task with `deleted_at` 91 days ago WHEN the purge cycle runs THEN its row and attachments are gone.
- **AC-RET-2** (INV-RET-5): GIVEN account deletion WHEN initiated THEN all refresh tokens are revoked and the session is invalid immediately.
- **AC-RET-3** (INV-RET-6): GIVEN account purge WHEN it completes THEN that user's tasks are owned by the workspace owner and still exist.
- **AC-RET-4** (INV-RET-4): GIVEN an admin WHEN attempting to delete an audit log entry THEN the operation is rejected.
- **AC-RET-5** (INV-RET-3): GIVEN an erasure request THEN completion timestamp is ≤ 30 days after request.

## Verify

```bash
npm test -- retention/purge        # INV-RET-1,2,6
npm test -- retention/account      # INV-RET-5
npm run test:int -- gdpr-erasure   # AC-RET-5
```
