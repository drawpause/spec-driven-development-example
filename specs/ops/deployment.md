# Deployment

```yaml
spec: ops/deployment
status: active
last_updated: 2026-06-14
owns:
  - .github/workflows/**
  - infra/ecs/**
  - Makefile
depends_on:
  - architecture
  - data/migrations
```

## Summary

How Relay ships: CI gates, environment promotion, zero-downtime ECS rollouts, migration ordering, rollback SLAs, and the health contract.

## Invariants

- **INV-DEPLOY-1:** Merges to `main` MUST pass CI (unit + integration tests, `tsc`, ESLint) and MUST auto-deploy to staging.
- **INV-DEPLOY-2:** Production deploys MUST be a manual promote (`make deploy-prod`) and MUST use a rolling ECS update (min 50% healthy, max 200% capacity).
- **INV-DEPLOY-3:** Migrations MUST run as a pre-deploy ECS task before new app tasks start (architecture INV-ARCH-5).
- **INV-DEPLOY-4:** Post-deploy health checks MUST poll `GET /health` and auto-rollback if any check is non-200.
- **INV-DEPLOY-5:** `GET /health` MUST require no auth and return 200 only when all dependency checks pass, else 503 with the failing check named.
- **INV-DEPLOY-6:** Secrets MUST come from AWS Secrets Manager at runtime; no secret values in repo, `.env`, or task definitions.
- **INV-DEPLOY-7:** Rollback MUST be initiable within 10 min of a confirmed incident and complete within 15 min.

## Contract

### Environments
| Env | URL | Branch | Trigger |
|---|---|---|---|
| `local` | `http://localhost:3000` | any | `make dev` |
| `staging` | `https://staging.relay.app` | `main` | auto on push to `main` |
| `production` | `https://relay.app` | `main` | manual `make deploy-prod` |

### Release process
1. PR → CI (tests, type check, lint).
2. Merge to `main` (1 approval + green CI) → auto-deploy staging.
3. Staging smoke test (`scripts/smoke-test.md`) + Playwright E2E.
4. `make deploy-prod` → rolling ECS deploy.
5. Health gate: poll `GET /health` every 10s for 2 min; auto-rollback on any non-200.

### Rollback
- Code: `make rollback` re-deploys previous task definition (< 5 min).
- DB: `make db-rollback MIGRATION={id}` applies the paired `.down.sql`.

### Health contract
```json
{ "status": "ok", "version": "0.5.0", "checks": { "database": "ok", "redis": "ok" } }
```
Any failing check → 503 with that check ≠ `ok`.

### Required env vars (values from Secrets Manager)
`DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `AWS_SES_FROM_ADDRESS`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `APP_BASE_URL`.

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-DEPLOY-1 | `.github/workflows/ci.yml` | `test`, `deploy-staging` jobs |
| INV-DEPLOY-2,4 | `infra/ecs/deploy.ts` | `rollingDeploy`, `healthGate` |
| INV-DEPLOY-3 | `infra/ecs/predeploy-migrate.ts` | `runMigrations` |
| INV-DEPLOY-5 | `src/server/routes/health.ts` | `health` |
| INV-DEPLOY-6 | `infra/ecs/task-def.ts` | `secretsFromManager` |

## Acceptance

- **AC-DEPLOY-1** (INV-DEPLOY-5): GIVEN Redis is down WHEN `GET /health` is called THEN response is 503 with `checks.redis != "ok"`.
- **AC-DEPLOY-2** (INV-DEPLOY-4): GIVEN a deploy whose new tasks fail health THEN the rollout auto-reverts to the prior task definition.
- **AC-DEPLOY-3** (INV-DEPLOY-3): GIVEN pending migrations WHEN deploying THEN the pre-deploy task completes before any new app task serves traffic.
- **AC-DEPLOY-4** (INV-DEPLOY-6): GIVEN the repo WHEN scanned THEN no secret value or `.env` with secrets is present.

## Verify

```bash
npm test -- routes/health        # INV-DEPLOY-5
npm run check:secrets            # INV-DEPLOY-6 (gitleaks)
npm run test:deploy -- --dry-run # INV-DEPLOY-2,3,4 against staging
```
