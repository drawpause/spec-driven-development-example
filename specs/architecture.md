# System Architecture

```yaml
spec: architecture
status: active
last_updated: 2026-06-14
owns:
  - src/server/**
  - src/workers/**
  - infra/**
depends_on:
  - overview
```

## Summary

Relay is an API-first monolith on AWS ECS (Node.js 20 + TypeScript) backed by PostgreSQL, Redis, and S3, with a React SPA client and background workers for all external I/O.

## Topology

```
Clients (React SPA, future mobile)
   │ HTTPS / WSS
API Server (Express REST + ws gateway)   ── ECS Fargate, autoscale 2–10
   ├── PostgreSQL 16 (RDS, Multi-AZ; read replica)
   ├── Redis 7 (ElastiCache; BullMQ queues + cache)
   └── S3 (uploads, presigned)
Background Workers (email/SES, webhooks, notifications) ── consume Redis queues
```

## Invariants

- **INV-ARCH-1:** All persistent mutations MUST go through the API server. No client and no worker writes user data via a path that bypasses the API's domain layer (`src/server/domain/**`).
- **INV-ARCH-2:** No outbound third-party HTTP call (SES, Stripe API) MUST occur in the request path; it MUST be enqueued and performed by a worker.
- **INV-ARCH-3:** The frontend MUST hold no privilege the public API does not; it authenticates with the same tokens as any API client.
- **INV-ARCH-4:** Code layout MUST match the `owns` globs: HTTP/domain in `src/server/**`, queue consumers in `src/workers/**`, IaC in `infra/**`.
- **INV-ARCH-5:** Schema migrations MUST run as a pre-deploy task before new app code starts, and MUST be backwards-compatible with the previous app version (see `data/migrations`, `ops/deployment`).
- **INV-ARCH-6:** Every environment in the Environments table MUST be provisioned only from `infra/**` (no manual console changes).

## Components

| Component | Runtime | Notes |
|---|---|---|
| API server | Node 20, Express, `ws` | ECS Fargate; autoscale 2–10 on CPU + queue depth |
| Database | PostgreSQL 16 (RDS Multi-AZ) | migrations via `node-pg-migrate`; read replica for analytics |
| Cache/queue | Redis 7 (ElastiCache) | BullMQ jobs; LRU cache for settings/flags, 5-min TTL |
| File storage | S3 + CloudFront | presigned uploads, max 25 MB, 10-min expiry |
| Frontend | React 18 + Tailwind | S3 + CloudFront; REST for mutations, WS for realtime |

## Environments

| Environment | Database | Notes |
|---|---|---|
| `local` | Docker Postgres | seeded fixtures; `make dev` |
| `staging` | RDS single-AZ | mirrors prod; reset weekly; auto-deploy on `main` |
| `production` | RDS Multi-AZ | read replica; manual promote |

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-ARCH-1 | `src/server/domain/**` | all mutators |
| INV-ARCH-2 | `src/server/jobs/enqueue.ts` | `enqueue` (only emitter in request path) |
| INV-ARCH-2 | `src/workers/**` | queue consumers |
| INV-ARCH-4 | repo layout | `src/server`, `src/workers`, `infra` |
| INV-ARCH-5 | `infra/ecs/predeploy-migrate.ts` | `runMigrations` |

## Acceptance

- **AC-ARCH-1** (INV-ARCH-2): GIVEN a request handler WHEN it is statically analyzed THEN it contains no direct `stripe.*` or `ses.*` network call; only `enqueue(...)`.
- **AC-ARCH-2** (INV-ARCH-1): GIVEN any DB write WHEN traced THEN it originates from `src/server/domain/**`.
- **AC-ARCH-3** (INV-ARCH-5): GIVEN a deploy WHEN migrations are pending THEN the pre-deploy task completes successfully before any new task receives traffic.

## Verify

```bash
npm run check:arch         # dependency-cruiser rules for INV-ARCH-1,2,4
npm test -- workers        # INV-ARCH-2 worker coverage
terraform -chdir=infra plan -detailed-exitcode   # INV-ARCH-6: no drift
```

For rationale on specific choices, see [DECISIONS.md](../DECISIONS.md).
