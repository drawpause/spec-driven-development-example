# Observability

```yaml
spec: ops/observability
status: active
last_updated: 2026-06-14
owns:
  - src/server/logging/**
  - src/server/metrics/**
  - infra/alerts/**
depends_on:
  - architecture
  - ops/security
```

## Summary

The logging, metrics, and alerting contract that makes Relay debuggable and SLO-enforced in production.

## Invariants

- **INV-OBS-1:** All logs MUST be structured JSON written to stdout and MUST include `timestamp, level, service, request_id, message` (plus `user_id` when authenticated).
- **INV-OBS-2:** Logs MUST NOT contain passwords, raw tokens, API keys, or full card numbers; email addresses MUST be masked (`j***@example.com`).
- **INV-OBS-3:** `debug` level MUST be disabled in production.
- **INV-OBS-4:** Every metric in the Metrics table MUST be emitted with its listed dimensions.
- **INV-OBS-5:** Every alert in the Alerting table MUST be configured with its condition, window, severity, and runbook link.
- **INV-OBS-6:** P1 alerts MUST page on-call via PagerDuty; all alerts MUST post to `#alerts`.

## Contract

### Required log fields
| Field | Type |
|---|---|
| `timestamp` | ISO-8601 |
| `level` | `error`/`warn`/`info`/`debug` |
| `service` | `api`/`worker`/`scheduler` |
| `request_id` | UUID (API-originated) |
| `user_id` | present if authenticated |
| `message` | string |

Level usage: `error` = unhandled/external failures; `warn` = recoverable (cache miss, retry); `info` = request lifecycle + business events; `debug` = internal state (prod-off, INV-OBS-3).

### Metrics
| Metric | Unit | Dimensions |
|---|---|---|
| `api.request.duration` | ms | `method`, `route`, `status_code` |
| `api.request.count` | count | `method`, `route`, `status_code` |
| `api.error.rate` | percent | `status_code` |
| `queue.job.duration` | ms | `queue_name` |
| `queue.job.failures` | count | `queue_name` |
| `queue.depth` | count | `queue_name` |
| `ws.connections.active` | count | — |
| `auth.failed_logins` | count | — |

### Alerting
| Alert | Condition | Window | Severity | Runbook |
|---|---|---|---|---|
| High API error rate | error rate > 1% | 5 min | P1 | `runbooks/high-error-rate.md` |
| High API latency | p95 > 1000ms | 5 min | P2 | `runbooks/high-latency.md` |
| Deep queue backlog | depth > 1000 | 10 min | P2 | `runbooks/worker-backlog.md` |
| DB CPU spike | RDS CPU > 80% | 10 min | P2 | `runbooks/db-cpu.md` |
| Credential stuffing | failed logins > 100/min | 1 min | P1 | `runbooks/credential-stuffing.md` |
| Health check failing | `/health` non-200 | 2 checks | P1 | `runbooks/health-check.md` |

Dashboards (CloudWatch `Relay-Production`): API Health, Background Jobs, Database, WebSockets, Auth.

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-OBS-1 | `src/server/logging/logger.ts` | `createLogger`, `BASE_FIELDS` |
| INV-OBS-2 | `src/server/logging/redact.ts` | `redact`, `maskEmail` |
| INV-OBS-3 | `src/server/logging/logger.ts` | `levelForEnv` |
| INV-OBS-4 | `src/server/metrics/emit.ts` | `METRICS` |
| INV-OBS-5,6 | `infra/alerts/*.tf` | alarm definitions |

## Acceptance

- **AC-OBS-1** (INV-OBS-1): GIVEN any log line WHEN parsed THEN it has all required fields.
- **AC-OBS-2** (INV-OBS-2): GIVEN a log call passing a token or email WHEN emitted THEN the token is absent and the email is masked.
- **AC-OBS-3** (INV-OBS-3): GIVEN `NODE_ENV=production` THEN `debug` logs are suppressed.
- **AC-OBS-4** (INV-OBS-5): GIVEN the alert config WHEN compared to the Alerting table THEN every alert exists with matching condition/window/severity/runbook.

## Verify

```bash
npm test -- logging/redact       # INV-OBS-2
npm test -- logging/fields       # INV-OBS-1,3
npm run check:alerts             # AC-OBS-4 (config ⇄ spec table)
```
