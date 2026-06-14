# Data Schema

```yaml
spec: data/schema
status: active
last_updated: 2026-06-14
database: PostgreSQL 16
owns:
  - src/db/schema.ts
  - migrations/**
depends_on:
  - overview
  - data/migrations
```

## Summary

The authoritative table definitions for Relay; the generated schema in `src/db/schema.ts` MUST match this document exactly.

## Invariants

- **INV-DATA-1:** All primary keys MUST be prefixed `text` IDs (`usr_`, `wsp_`, `prj_`, `tsk_`, `notif_`, `sub_`, `rt_`), not `uuid`.
- **INV-DATA-2:** The live schema MUST match the tables below: a column added/removed/retyped in code without a corresponding edit here is a spec violation.
- **INV-DATA-3:** Every foreign key listed MUST exist with `ON DELETE` behavior consistent with `data/retention`.
- **INV-DATA-4:** `users.email` MUST be UNIQUE and stored lowercased/normalized.
- **INV-DATA-5:** Soft-deletable rows (`tasks`, `workspaces`) MUST use a nullable `deleted_at` and MUST be excluded from default queries.
- **INV-DATA-6:** `tasks.status` MUST be constrained to the enum in `overview` INV-CORE-3.
- **INV-DATA-7:** `subscriptions` MUST have exactly one row per workspace (`workspace_id` UNIQUE) and is written only per `features/payments` INV-PAY-1.

## Contract

### users
| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `text` | PK | `usr_*` |
| `email` | `text` | UNIQUE, NOT NULL | lowercased on write |
| `name` | `text` | NOT NULL | |
| `password_hash` | `text` | NULL | null for SSO-only users |
| `google_id` | `text` | UNIQUE, NULL | Google `sub` claim |
| `email_verified_at` | `timestamptz` | NULL | null until verified |
| `failed_login_attempts` | `int` | NOT NULL DEFAULT 0 | reset on success |
| `locked_until` | `timestamptz` | NULL | non-null while locked |
| `notification_preferences` | `jsonb` | NOT NULL DEFAULT '{}' | event→channel toggles |
| `created_at` | `timestamptz` | NOT NULL DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL | on every write |

`notification_preferences` shape (keys = `features/notifications` Event Catalog; missing keys use catalog defaults):
```json
{ "task_assigned": {"in_app":true,"email":true}, "mention": {"in_app":true,"email":true},
  "task_completed": {"in_app":true,"email":false}, "due_reminder": {"in_app":true,"email":true},
  "member_added": {"in_app":false,"email":false} }
```

### workspaces
| Column | Type | Constraints |
|---|---|---|
| `id` | `text` | PK `wsp_*` |
| `name` | `text` | NOT NULL |
| `slug` | `text` | UNIQUE, NOT NULL |
| `owner_id` | `text` | FK→users.id, NOT NULL |
| `plan` | `text` | NOT NULL DEFAULT 'free' (`free`/`pro`/`enterprise`) |
| `stripe_customer_id` | `text` | UNIQUE, NULL |
| `deleted_at` | `timestamptz` | NULL |
| `created_at` | `timestamptz` | NOT NULL |

### projects
| Column | Type | Constraints |
|---|---|---|
| `id` | `text` | PK `prj_*` |
| `workspace_id` | `text` | FK→workspaces.id, NOT NULL |
| `name` | `text` | NOT NULL |
| `description` | `text` | NULL |
| `due_at` | `timestamptz` | NULL |
| `archived_at` | `timestamptz` | NULL |
| `created_by` | `text` | FK→users.id, NOT NULL |
| `created_at` | `timestamptz` | NOT NULL |

### tasks
| Column | Type | Constraints |
|---|---|---|
| `id` | `text` | PK `tsk_*` |
| `project_id` | `text` | FK→projects.id, NOT NULL |
| `title` | `text` | NOT NULL, ≤ 500 chars |
| `description` | `text` | NULL, markdown, ≤ 10000 chars |
| `status` | `text` | NOT NULL DEFAULT 'todo' (enum INV-CORE-3) |
| `assignee_id` | `text` | FK→users.id, NULL |
| `due_at` | `timestamptz` | NULL |
| `deleted_at` | `timestamptz` | NULL |
| `created_by` | `text` | FK→users.id, NOT NULL |
| `created_at` / `updated_at` | `timestamptz` | NOT NULL |

### notifications
| Column | Type | Constraints |
|---|---|---|
| `id` | `text` | PK `notif_*` |
| `user_id` | `text` | FK→users.id, NOT NULL (recipient) |
| `event` | `text` | NOT NULL (Event Catalog key) |
| `payload` | `jsonb` | NOT NULL |
| `read_at` | `timestamptz` | NULL = unread |
| `created_at` | `timestamptz` | NOT NULL |

### subscriptions (mirror of Stripe; webhook-only writes)
| Column | Type | Constraints |
|---|---|---|
| `id` | `text` | PK (Stripe `sub_*`) |
| `workspace_id` | `text` | FK→workspaces.id, UNIQUE |
| `plan` | `text` | NOT NULL |
| `status` | `text` | NOT NULL (`active`/`past_due`/`canceled`/`trialing`) |
| `seats` | `int` | NOT NULL |
| `current_period_end` | `timestamptz` | NOT NULL |
| `updated_at` | `timestamptz` | NOT NULL |

### refresh_tokens
| Column | Type | Constraints |
|---|---|---|
| `id` | `text` | PK `rt_*` |
| `user_id` | `text` | FK→users.id, NOT NULL |
| `token_hash` | `text` | UNIQUE, NOT NULL (bcrypt) |
| `expires_at` | `timestamptz` | NOT NULL (30d) |
| `revoked_at` | `timestamptz` | NULL |
| `created_at` | `timestamptz` | NOT NULL |

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-DATA-1,2 | `src/db/schema.ts` | table definitions |
| INV-DATA-4 | `src/db/schema.ts` | `users.email` unique index + `normalizeEmail` |
| INV-DATA-5 | `src/db/scopes.ts` | `notDeleted` default scope |
| INV-DATA-6 | `migrations/*` | `tasks_status_check` constraint |
| INV-DATA-7 | `src/db/schema.ts` | `subscriptions.workspace_id` unique |

## Acceptance

- **AC-DATA-1** (INV-DATA-2): GIVEN the live DB WHEN introspected THEN it is structurally equal to this document (column names, types, nullability, constraints).
- **AC-DATA-2** (INV-DATA-6): GIVEN an INSERT with `tasks.status = 'archived'` THEN the DB rejects it via check constraint.
- **AC-DATA-3** (INV-DATA-5): GIVEN a soft-deleted task WHEN queried via the default scope THEN it is not returned.
- **AC-DATA-4** (INV-DATA-1): GIVEN any new row WHEN its id is read THEN it carries the entity prefix.

## Verify

```bash
npm run db:diff -- --against spec   # AC-DATA-1: introspection vs this spec
npm test -- db/constraints          # INV-DATA-4,6,7
npm test -- db/scopes               # INV-DATA-5
```
