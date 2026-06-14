# Deployment

**Last Updated:** 2024-11-01

---

## Environments

| Environment | URL | Branch | Deploy Trigger |
|---|---|---|---|
| `local` | `http://localhost:3000` | any | Manual (`make dev`) |
| `staging` | `https://staging.relay.app` | `main` | Automatic on push to `main` |
| `production` | `https://relay.app` | `main` | Manual promote via `make deploy-prod` |

---

## Release Process

1. **Open a PR** against `main`. CI runs: unit tests, integration tests, TypeScript type check, ESLint.
2. **Merge to `main`** (requires 1 approval + passing CI). This automatically deploys to staging.
3. **Staging verification:** The merging engineer runs a smoke test checklist (see `scripts/smoke-test.md`). The automated E2E suite (Playwright) also runs against staging.
4. **Promote to production:** Any engineer with deploy access runs `make deploy-prod`. This triggers a rolling ECS deployment.
5. **Post-deploy health check:** Automated script pings `GET /health` every 10 seconds for 2 minutes. Rolls back automatically if any check returns non-200.

---

## Zero-Downtime Deploys

- ECS rolling update: minimum 50% healthy instances, maximum 200% capacity during the rollout.
- Database migrations run as a separate pre-deploy ECS task, before the new app version starts.
- Migrations must always be backwards-compatible with the previous app version (old code + new schema must work).

---

## Rollback

**Code rollback:** `make rollback` re-deploys the previous ECS task definition. Takes < 5 minutes.

**Database rollback:** Each migration must have a `{name}.down.sql` file. Run `make db-rollback MIGRATION=20241101_001` to apply the down migration.

**SLA:** A rollback must be initiated within 10 minutes of a confirmed incident and completed within 15 minutes.

---

## Health Check Endpoint

`GET /health` returns `200 OK` with no auth required:

```json
{
  "status": "ok",
  "version": "0.4.0",
  "checks": {
    "database": "ok",
    "redis": "ok"
  }
}
```

If any check fails, it returns `503 Service Unavailable` with the failing check identified.

---

## Environment Variables

All secrets are stored in AWS Secrets Manager and injected into ECS tasks at runtime. They are never stored in `.env` files committed to the repository.

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `JWT_SECRET` | HMAC secret for signing JWTs |
| `STRIPE_SECRET_KEY` | Stripe API key |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook endpoint signing secret |
| `AWS_SES_FROM_ADDRESS` | From address for transactional email (e.g., `noreply@relay.app`) |
| `GOOGLE_CLIENT_ID` | Google OAuth 2.0 client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth 2.0 client secret |
| `APP_BASE_URL` | Public base URL (e.g., `https://relay.app`) for constructing links in emails |
