# UI Copy

**Last Updated:** 2024-11-01

---

## Voice and Tone

- **Clear over clever.** No jargon, no puns in functional UI copy.
- **Brief.** Button labels are 1-3 words. Error messages are one sentence.
- **Human.** First person for the user ("Your tasks"), second person for the product voice ("You've been assigned…").
- **Calm on failure.** Errors don't blame the user. They state what happened and what to do next.

---

## Auth

| Location | Copy |
|---|---|
| Sign-up headline | "Get your team organized" |
| Sign-up CTA button | "Get started free" |
| Login CTA button | "Log in" |
| Google SSO button | "Continue with Google" |
| "Forgot password" link | "Forgot your password?" |
| Password reset email subject | "Reset your Relay password" |
| Verification email subject | "Confirm your email address" |
| Verification pending message | "Check your inbox to verify your email." |
| Resend verification CTA | "Resend verification email" |

---

## Errors

| Error code | User-facing message |
|---|---|
| `INVALID_CREDENTIALS` | "Incorrect email or password." |
| `EMAIL_NOT_VERIFIED` | "Please verify your email before logging in. Check your inbox or resend the link." |
| `ACCOUNT_LOCKED` | "Too many login attempts. Try again in {N} minutes." |
| `PLAN_LIMIT_EXCEEDED` (projects) | "You've reached the project limit on the Free plan. Upgrade to Pro for unlimited projects." |
| `PLAN_LIMIT_EXCEEDED` (members) | "You've reached the member limit. Upgrade to Pro to add more members." |
| `VALIDATION_ERROR` (generic) | "Please fix the errors below." |
| `RATE_LIMITED` | "Too many requests. Please wait a moment and try again." |
| Server error (500) | "Something went wrong. We've been notified and are looking into it." |

---

## Empty States

| Context | Heading | Body | CTA label |
|---|---|---|---|
| No tasks in project | "No tasks yet" | "Add your first task to get started." | "Add task" |
| No notifications | "You're all caught up" | *(none)* | *(none)* |
| No projects in workspace | "No projects yet" | "Create a project to start organizing your work." | "New project" |
| No members in project | "Just you so far" | "Invite teammates to collaborate on this project." | "Invite people" |
| No search results | "No results for "{query}"" | "Try a different search term." | *(none)* |

---

## Confirmation and Destructive Actions

All destructive actions require an explicit confirmation dialog. The primary action button uses a danger color.

| Action | Dialog title | Dialog body | Cancel label | Confirm label |
|---|---|---|---|---|
| Delete task | "Delete task?" | "This can't be undone." | "Cancel" | "Delete task" |
| Archive project | "Archive project?" | "Members will lose access until you unarchive it." | "Cancel" | "Archive project" |
| Cancel subscription | "Cancel your Pro plan?" | "You'll keep access to Pro features until {date}." | "Keep plan" | "Cancel subscription" |
| Delete workspace | "Delete {workspace name}?" | "All projects and tasks will be permanently removed after 30 days. This cannot be undone." | "Cancel" | "Delete workspace" |

---

## Notification Copy

### In-App Notifications

| Event | Notification text |
|---|---|
| Task assigned | "{sender name} assigned you "{task title}"" |
| @mention | "{sender name} mentioned you in "{task title}"" |
| Task completed | ""{task title}" was marked done" |
| Due date reminder | ""{task title}" is due tomorrow" |
| Added to project | "{sender name} added you to "{project name}"" |

### Email Subjects

| Event | Subject line |
|---|---|
| Task assigned | "You've been assigned: {task title}" |
| @mention | "{sender name} mentioned you in Relay" |
| Due date reminder | "Reminder: {task title} is due tomorrow" |
| Invoice paid | "Your Relay invoice is ready" |
| Payment failed | "Action required: payment failed for Relay" |
| Subscription cancelled | "Your Relay Pro subscription has been cancelled" |
