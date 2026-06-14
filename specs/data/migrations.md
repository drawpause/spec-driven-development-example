# Database Migrations

```yaml
spec: data/migrations
status: active
last_updated: 2026-06-14
owns:
  - migrations/**
  - scripts/backfill-*.ts
depends_on:
  - data/schema
  - ops/deployment
```

## Summary

The policy and history for evolving the Relay schema with zero-downtime, backwards-compatible migrations.

## Invariants

- **INV-MIG-1:** Every migration MUST be backwards-compatible with the previously deployed app version (old code + new schema MUST work) for at least one deploy cycle.
- **INV-MIG-2:** A column MUST NOT be renamed or dropped in a single migration: add → backfill → switch code → drop in a later migration.
- **INV-MIG-3:** Migrations MUST NOT take long table locks: use `CREATE INDEX CONCURRENTLY`; never `ALTER TABLE ... ADD COLUMN NOT NULL` without a default.
- **INV-MIG-4:** Every forward migration `migrations/{ts}_{desc}.sql` MUST have a paired `migrations/{ts}_{desc}.down.sql`.
- **INV-MIG-5:** Any schema change MUST be reflected in `data/schema` in the same PR (review-enforced).
- **INV-MIG-6:** A migration with a backfill prerequisite MUST NOT apply its constraint until the named backfill script has run and been verified in staging.

## Contract

File naming:
```
migrations/{timestamp}_{NNN}_{description}.sql
migrations/{timestamp}_{NNN}_{description}.down.sql
```

### Migration log
| ID | Description |
|---|---|
| `20240701_001_create_users` | Initial `users`. |
| `20240701_002_create_workspaces_projects_tasks` | Initial `workspaces`, `projects`, `tasks`, `refresh_tokens`. |
| `20240801_001_add_stripe_customer_id_to_workspaces` | `stripe_customer_id text NULL` (nullable; NOT NULL added after backfill). |
| `20240815_001_add_not_null_stripe_customer_id` | NOT NULL on `workspaces.stripe_customer_id` after backfill. Prereq: `scripts/backfill-stripe-customers.ts`. |
| `20240901_001_create_subscriptions` | `subscriptions` table. |
| `20241101_001_add_notifications` | `notifications` table; `notification_preferences jsonb DEFAULT '{}'` and `google_id text UNIQUE NULL` on `users`. |

### Backfill procedure (template — 20240815)
1. `npx ts-node scripts/backfill-stripe-customers.ts --dry-run` and inspect.
2. Run without `--dry-run` in staging; verify.
3. Run in production off-peak.
4. Apply the NOT NULL migration only after all rows populated.

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-MIG-1,3 | `scripts/lint-migration.ts` | `assertBackwardsCompatible`, `assertNoBlockingLock` |
| INV-MIG-4 | `scripts/lint-migration.ts` | `assertDownExists` |
| INV-MIG-5 | CI | `check:schema-drift` |
| INV-MIG-6 | `migrations/20240815_001_*.sql` | guarded by backfill checkpoint |

## Acceptance

- **AC-MIG-1** (INV-MIG-4): GIVEN any `*.sql` forward migration WHEN scanned THEN a matching `*.down.sql` exists.
- **AC-MIG-2** (INV-MIG-3): GIVEN a migration containing `ADD COLUMN ... NOT NULL` without `DEFAULT` or a non-concurrent `CREATE INDEX` THEN the linter fails.
- **AC-MIG-3** (INV-MIG-1): GIVEN the previous app image WHEN run against the new schema in CI THEN its test suite passes.
- **AC-MIG-4** (INV-MIG-5): GIVEN a migration touching a column WHEN `data/schema` is unchanged in the PR THEN `check:schema-drift` fails.

## Verify

```bash
npm run lint:migrations         # INV-MIG-3,4
npm run test:compat -- --prev   # AC-MIG-3: old app vs new schema
npm run check:schema-drift      # AC-MIG-4
```
