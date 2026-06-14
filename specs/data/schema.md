# Data Schema

**Last Updated:** 2024-11-01
**Database:** PostgreSQL 16

---

## ID Convention

All IDs are prefixed, randomly-generated strings (e.g., `usr_j3k9x`, `tsk_m2p1z`). They are stored as `text`, not `uuid`, to make them human-readable in logs and URLs.

---

## Tables

### users

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `text` | PK | Prefixed: `usr_*` |
| `email` | `text` | UNIQUE, NOT NULL | Lowercased and normalized on write |
| `name` | `text` | NOT NULL | Display name |
| `password_hash` | `text` | NULLABLE | Null if the user only uses Google SSO |
| `google_id` | `text` | UNIQUE, NULLABLE | Google OAuth `sub` claim |
| `email_verified_at` | `timestamptz` | NULLABLE | Null until the user clicks the verification link |
| `failed_login_attempts` | `int` | NOT NULL, DEFAULT 0 | Reset to 0 on successful login |
| `locked_until` | `timestamptz` | NULLABLE | Non-null while account is locked out |
| `notification_preferences` | `jsonb` | NOT NULL, DEFAULT '{}' | Map of event type to channel toggles |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL | Updated on every write |

**notification_preferences shape:**
```json
{
  "task_assigned":   { "in_app": true, "email": true },
  "mention":         { "in_app": true, "email": true },
  "task_completed":  { "in_app": true, "email": false },
  "due_reminder":    { "in_app": true, "email": true },
  "member_added":    { "in_app": false, "email": false }
}
```
Missing keys use the default values defined in `specs/features/notifications.md`.

---

### workspaces

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `text` | PK | `wsp_*` |
| `name` | `text` | NOT NULL | Display name |
| `slug` | `text` | UNIQUE, NOT NULL | URL-safe identifier (e.g., `acme-corp`) |
| `owner_id` | `text` | FK -> users.id, NOT NULL | |
| `plan` | `text` | NOT NULL, DEFAULT 'free' | `free`, `pro`, or `enterprise` |
| `stripe_customer_id` | `text` | UNIQUE, NULLABLE | Set when a Stripe customer is created |
| `deleted_at` | `timestamptz` | NULLABLE | Soft delete |
| `created_at` | `timestamptz` | NOT NULL | |

---

### projects

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `text` | PK | `prj_*` |
| `workspace_id` | `text` | FK -> workspaces.id, NOT NULL | |
| `name` | `text` | NOT NULL | |
| `description` | `text` | NULLABLE | Plain text |
| `due_at` | `timestamptz` | NULLABLE | |
| `archived_at` | `timestamptz` | NULLABLE | Archived projects are hidden from default views |
| `created_by` | `text` | FK -> users.id, NOT NULL | |
| `created_at` | `timestamptz` | NOT NULL | |

---

### tasks

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `text` | PK | `tsk_*` |
| `project_id` | `text` | FK -> projects.id, NOT NULL | |
| `title` | `text` | NOT NULL | Max 500 characters |
| `description` | `text` | NULLABLE | Markdown, max 10,000 characters |
| `status` | `text` | NOT NULL, DEFAULT 'todo' | See glossary in overview.md |
| `assignee_id` | `text` | FK -> users.id, NULLABLE | |
| `due_at` | `timestamptz` | NULLABLE | |
| `deleted_at` | `timestamptz` | NULLABLE | Soft delete |
| `created_by` | `text` | FK -> users.id, NOT NULL | |
| `created_at` | `timestamptz` | NOT NULL | |
| `updated_at` | `timestamptz` | NOT NULL | |

---

### notifications

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `text` | PK | `notif_*` |
| `user_id` | `text` | FK -> users.id, NOT NULL | The recipient |
| `event` | `text` | NOT NULL | `task_assigned`, `mention`, `task_completed`, `due_reminder`, `member_added` |
| `payload` | `jsonb` | NOT NULL | Event-specific context (task ID, title, sender, etc.) |
| `read_at` | `timestamptz` | NULLABLE | Null means unread |
| `created_at` | `timestamptz` | NOT NULL | |

---

### subscriptions

Mirrors Stripe subscription state. Updated exclusively via webhooks.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `text` | PK | The Stripe subscription ID (`sub_*`) |
| `workspace_id` | `text` | FK -> workspaces.id, UNIQUE | One subscription per workspace |
| `plan` | `text` | NOT NULL | `free`, `pro`, or `enterprise` |
| `status` | `text` | NOT NULL | Mirrors Stripe: `active`, `past_due`, `canceled`, `trialing` |
| `seats` | `int` | NOT NULL | Number of purchased seats |
| `current_period_end` | `timestamptz` | NOT NULL | When the current billing period ends |
| `updated_at` | `timestamptz` | NOT NULL | Updated on each webhook |

---

### refresh_tokens

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `text` | PK | `rt_*` |
| `user_id` | `text` | FK -> users.id, NOT NULL | |
| `token_hash` | `text` | UNIQUE, NOT NULL | bcrypt hash of the raw token |
| `expires_at` | `timestamptz` | NOT NULL | 30 days from issuance |
| `revoked_at` | `timestamptz` | NULLABLE | Set on logout, rotation, or password reset |
| `created_at` | `timestamptz` | NOT NULL | |
