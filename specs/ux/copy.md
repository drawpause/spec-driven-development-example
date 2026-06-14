# UI Copy

```yaml
spec: ux/copy
status: active
last_updated: 2026-06-14
owns:
  - src/web/i18n/en.json
  - src/web/copy/**
depends_on:
  - api/errors
  - features/notifications
```

## Summary

The authoritative string catalog for Relay's UI; `src/web/i18n/en.json` MUST contain exactly these keys and values.

## Invariants

- **INV-COPY-1:** Every string the UI renders MUST come from a catalog key below; no hardcoded literals in components.
- **INV-COPY-2:** Each error `code` in `api/errors` that surfaces to users MUST have a corresponding user-facing message here.
- **INV-COPY-3:** Placeholders in `{braces}` MUST be preserved verbatim by any translation/edit (count and names unchanged).
- **INV-COPY-4:** Notification copy keys MUST match the `features/notifications` Event Catalog keys 1:1.
- **INV-COPY-5:** Voice rules MUST hold: button labels 1–3 words; error messages one sentence; failures state cause + next step, never blame the user.

## Contract

### Auth
| Key | Value |
|---|---|
| `auth.signup.headline` | "Get your team organized" |
| `auth.signup.cta` | "Get started free" |
| `auth.login.cta` | "Log in" |
| `auth.google.cta` | "Continue with Google" |
| `auth.forgot.link` | "Forgot your password?" |
| `auth.reset.email_subject` | "Reset your Relay password" |
| `auth.verify.email_subject` | "Confirm your email address" |
| `auth.verify.pending` | "Check your inbox to verify your email." |
| `auth.verify.resend_cta` | "Resend verification email" |

### Errors (keyed by `api/errors` code — INV-COPY-2)
| Key (code) | Value |
|---|---|
| `INVALID_CREDENTIALS` | "Incorrect email or password." |
| `EMAIL_NOT_VERIFIED` | "Please verify your email before logging in. Check your inbox or resend the link." |
| `ACCOUNT_LOCKED` | "Too many login attempts. Try again in {minutes} minutes." |
| `PLAN_LIMIT_EXCEEDED.projects` | "You've reached the project limit on the Free plan. Upgrade to Pro for unlimited projects." |
| `PLAN_LIMIT_EXCEEDED.members` | "You've reached the member limit. Upgrade to Pro to add more members." |
| `VALIDATION_ERROR.generic` | "Please fix the errors below." |
| `RATE_LIMITED` | "Too many requests. Please wait a moment and try again." |
| `INTERNAL.500` | "Something went wrong. We've been notified and are looking into it." |

### Empty states
| Key | Heading | Body | CTA |
|---|---|---|---|
| `empty.tasks` | "No tasks yet" | "Add your first task to get started." | "Add task" |
| `empty.notifications` | "You're all caught up" | — | — |
| `empty.projects` | "No projects yet" | "Create a project to start organizing your work." | "New project" |
| `empty.members` | "Just you so far" | "Invite teammates to collaborate on this project." | "Invite people" |
| `empty.search` | "No results for \"{query}\"" | "Try a different search term." | — |

### Destructive confirmations
| Key | Title | Body | Cancel | Confirm |
|---|---|---|---|---|
| `confirm.delete_task` | "Delete task?" | "This can't be undone." | "Cancel" | "Delete task" |
| `confirm.archive_project` | "Archive project?" | "Members will lose access until you unarchive it." | "Cancel" | "Archive project" |
| `confirm.cancel_sub` | "Cancel your Pro plan?" | "You'll keep access to Pro features until {date}." | "Keep plan" | "Cancel subscription" |
| `confirm.delete_workspace` | "Delete {workspace}?" | "All projects and tasks will be permanently removed after 30 days. This cannot be undone." | "Cancel" | "Delete workspace" |

### Notifications (keys = Event Catalog — INV-COPY-4)
| Key | In-app text | Email subject |
|---|---|---|
| `task_assigned` | "{sender} assigned you \"{task}\"" | "You've been assigned: {task}" |
| `mention` | "{sender} mentioned you in \"{task}\"" | "{sender} mentioned you in Relay" |
| `task_completed` | "\"{task}\" was marked done" | — |
| `due_reminder` | "\"{task}\" is due tomorrow" | "Reminder: {task} is due tomorrow" |
| `member_added` | "{sender} added you to \"{project}\"" | — |
| `invoice_paid` | — | "Your Relay invoice is ready" |
| `payment_failed` | — | "Action required: payment failed for Relay" |
| `subscription_cancelled` | — | "Your Relay Pro subscription has been cancelled" |

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-COPY-1 | `src/web/i18n/en.json` | catalog |
| INV-COPY-1 | `src/web/copy/t.ts` | `t(key, vars)` |
| INV-COPY-2,4 | `scripts/check-copy.ts` | `assertCoverage` |

## Acceptance

- **AC-COPY-1** (INV-COPY-2): GIVEN the error code set WHEN compared to the Errors table THEN every user-facing code has a message.
- **AC-COPY-2** (INV-COPY-1): GIVEN the web bundle WHEN scanned THEN no JSX text literal exists outside the catalog.
- **AC-COPY-3** (INV-COPY-3): GIVEN any value WHEN edited THEN its `{placeholders}` are unchanged in name and count.
- **AC-COPY-4** (INV-COPY-4): GIVEN notification keys WHEN compared to the Event Catalog THEN they match 1:1.

## Verify

```bash
npm run check:copy          # INV-COPY-2,3,4 coverage + placeholder integrity
npm run lint:no-literals    # INV-COPY-1
```
