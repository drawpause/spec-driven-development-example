# UX Flows

```yaml
spec: ux/flows
status: active
last_updated: 2026-06-14
owns:
  - src/web/flows/**
  - src/web/state/**
depends_on:
  - features/auth
  - features/notifications
  - features/payments
  - overview
```

## Summary

The deterministic client-side flows and state machines for onboarding, task lifecycle, the notification inbox, and billing upgrade — expressed as steps and transitions, not narrative.

## Invariants

- **INV-FLOW-1:** The task status machine MUST permit only the transitions in the table below; all transitions are explicit user actions (no automatic moves).
- **INV-FLOW-2:** Any status MUST be able to transition to `cancelled`; `done` and `cancelled` MUST be re-openable to `todo`.
- **INV-FLOW-3:** Every status change MUST append a task activity entry.
- **INV-FLOW-4:** Onboarding MUST resume at the earliest incomplete step on next login if abandoned.
- **INV-FLOW-5:** The unread badge MUST reflect the real-time unread count (notifications INV-NOTIF-7) and MUST be hidden when count is 0.
- **INV-FLOW-6:** Notification preference toggles MUST auto-save on change (no explicit save action).
- **INV-FLOW-7:** Seat selection in the upgrade flow MUST enforce `min = current member count`.
- **INV-FLOW-8:** On payment failure during checkout, the modal MUST stay open and show the gateway error inline.

## Contract

### Onboarding (direct signup) — ordered steps
`signup → email_verification_pending → create_workspace → invite_teammates(skippable) → create_first_project → empty_project_view`

Edge cases:
| Condition | Behavior |
|---|---|
| Verification link expired (>24h) | show "Link expired" + resend action |
| Workspace slug taken | inline error + up to 3 auto-generated alternatives |
| Tab closed mid-flow | resume at earliest incomplete step (INV-FLOW-4) |

### Task status machine (INV-FLOW-1)
| From | Allowed → |
|---|---|
| `todo` | `in_progress`, `cancelled` |
| `in_progress` | `in_review`, `cancelled` |
| `in_review` | `done`, `cancelled` |
| `done` | `todo` (re-open), `cancelled` |
| `cancelled` | `todo` (re-open) |

### Notification inbox
- Bell badge: real-time unread count; hidden at 0 (INV-FLOW-5).
- Panel: slides from right; newest-first; day groups `Today` / `Yesterday` / `Earlier`.
- `Mark all read`: top-right; calls `POST /notifications/read-all`.
- Row click navigates to the related task.
- Empty state heading: `You're all caught up` (copy owned by `ux/copy`).

### Billing upgrade
`billing_panel → upgrade_modal(select seats, min=members) → payment(Stripe Elements) → success`
- Total = `seats × $12` for Pro.
- Failure → INV-FLOW-8.

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-FLOW-1,2,3 | `src/web/state/taskStatus.ts` | `TASK_TRANSITIONS`, `transition` |
| INV-FLOW-4 | `src/web/flows/onboarding.ts` | `resumeStep` |
| INV-FLOW-5 | `src/web/flows/inbox.ts` | `unreadBadge` |
| INV-FLOW-6 | `src/web/flows/preferences.ts` | `onToggle` (debounced save) |
| INV-FLOW-7,8 | `src/web/flows/upgrade.ts` | `seatMin`, `onPaymentError` |

## Acceptance

- **AC-FLOW-1** (INV-FLOW-1): GIVEN a `todo` task WHEN moved to `in_review` THEN the transition is rejected (not adjacent).
- **AC-FLOW-2** (INV-FLOW-2): GIVEN a `done` task WHEN re-opened THEN status becomes `todo` and an activity entry is added.
- **AC-FLOW-3** (INV-FLOW-4): GIVEN onboarding abandoned after `create_workspace` WHEN the user logs in again THEN they land on `invite_teammates`.
- **AC-FLOW-4** (INV-FLOW-7): GIVEN a workspace with 6 members WHEN opening the upgrade modal THEN seat selector minimum is 6.
- **AC-FLOW-5** (INV-FLOW-8): GIVEN a declined card WHEN submitting THEN the modal remains open with the gateway message visible.

## Verify

```bash
npm test -- web/state/taskStatus     # INV-FLOW-1,2,3
npm run test:e2e -- onboarding        # AC-FLOW-3
npm run test:e2e -- upgrade           # AC-FLOW-4,5
```
