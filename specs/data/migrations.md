# Database Migrations

**Last Updated:** 2024-11-01

---

## Policy

1. All schema changes must be reflected in `specs/data/schema.md` and reviewed before a migration file is written.
2. Migrations must be backwards-compatible with the previous deployed version of the application for at least one deploy cycle. This enables zero-downtime deploys.
3. Never rename or drop a column in a single migration. Add the new column first, backfill data, update application code, then remove the old column in a follow-up migration.
4. Never use operations that take long table locks in production. Use `CREATE INDEX CONCURRENTLY` and avoid `ALTER TABLE ... ADD COLUMN NOT NULL` without a default.

---

## File Naming

```
migrations/{timestamp}_{description}.sql
```

Example:
```
migrations/
  20240701_001_create_users.sql
  20240701_002_create_workspaces_projects_tasks.sql
  20240801_001_add_stripe_customer_id_to_workspaces.sql
```

Each migration file must also have a corresponding rollback file:
```
migrations/20240701_001_create_users.down.sql
```

---

## Migration Log

### 20240701_001_create_users
**Description:** Initial `users` table.
**Columns added:** `id`, `email`, `name`, `password_hash`, `email_verified_at`, `failed_login_attempts`, `locked_until`, `created_at`, `updated_at`.

### 20240701_002_create_workspaces_projects_tasks
**Description:** Initial `workspaces`, `projects`, `tasks`, and `refresh_tokens` tables.

### 20240801_001_add_stripe_customer_id_to_workspaces
**Description:** Added `stripe_customer_id text NULLABLE` to `workspaces`.
**Backwards-compatible:** Added as nullable. NOT NULL constraint added after backfill (see 20240815_001).

### 20240815_001_add_not_null_stripe_customer_id
**Description:** Added NOT NULL constraint to `workspaces.stripe_customer_id` after backfill completed.
**Prerequisite:** `scripts/backfill-stripe-customers.ts` must be run and verified in staging before applying to production.

### 20240901_001_create_subscriptions
**Description:** New `subscriptions` table for mirroring Stripe state.

### 20241101_001_add_notifications
**Description:** Added `notifications` table and `notification_preferences jsonb DEFAULT '{}'` to `users`.
**Added:** `google_id text UNIQUE NULLABLE` to `users` (for Google SSO).

---

## Backfill Procedures

### 20240815 — Backfill Stripe customer IDs
After the nullable column was added, all existing workspaces needed a Stripe customer created.

1. Run `npx ts-node scripts/backfill-stripe-customers.ts --dry-run` and inspect output.
2. Run without `--dry-run` in staging and verify.
3. Run in production during off-peak hours.
4. Once all rows are populated, apply `20240815_001_add_not_null_stripe_customer_id.sql`.
