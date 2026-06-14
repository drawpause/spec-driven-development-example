# Observability

**Last Updated:** 2024-11-01

---

## Logging

### Format and Collection
All logs are structured JSON, written to stdout. Collected by the ECS awslogs log driver and shipped to CloudWatch Logs.

### Required Fields on Every Log Line

| Field | Type | Description |
|---|---|---|
| `timestamp` | ISO 8601 | When the event occurred |
| `level` | string | `error`, `warn`, `info`, or `debug` |
| `service` | string | `api`, `worker`, or `scheduler` |
| `request_id` | string | UUID, present on all API-originated logs |
| `user_id` | string | Authenticated user ID, if applicable |
| `message` | string | Human-readable summary |

### Log Levels

| Level | When to use |
|---|---|
| `error` | Unhandled exceptions, failed external service calls |
| `warn` | Recoverable issues (cache miss, retry attempt, degraded response) |
| `info` | Request lifecycle (start/end), significant business events (user signed up, payment processed) |
| `debug` | Detailed internal state. **Disabled in production.** |

### Sensitive Data
Never log: passwords, raw tokens, API keys, or full credit card numbers. Email addresses may appear in `info`-level request logs but must be masked: `j***@example.com`.

---

## Metrics

Emitted to CloudWatch Metrics using the `aws-embedded-metrics` library.

| Metric name | Unit | Dimensions | Description |
|---|---|---|---|
| `api.request.duration` | Milliseconds | `method`, `route`, `status_code` | Per-endpoint latency |
| `api.request.count` | Count | `method`, `route`, `status_code` | Request volume |
| `api.error.rate` | Percent | `status_code` | Rolling error rate |
| `queue.job.duration` | Milliseconds | `queue_name` | Background job processing time |
| `queue.job.failures` | Count | `queue_name` | Failed jobs |
| `queue.depth` | Count | `queue_name` | Pending jobs in queue |
| `ws.connections.active` | Count | — | Current active WebSocket connections |
| `auth.failed_logins` | Count | — | Failed login attempts per minute |

---

## Alerting

Alerts route to `#alerts` in Slack for all severities. P1 alerts additionally page the on-call engineer via PagerDuty.

| Alert | Condition | Window | Severity | Runbook |
|---|---|---|---|---|
| High API error rate | Error rate > 1% | 5 min sustained | P1 | `runbooks/high-error-rate.md` |
| High API latency | p95 > 1000ms | 5 min sustained | P2 | `runbooks/high-latency.md` |
| Deep queue backlog | Queue depth > 1000 | 10 min sustained | P2 | `runbooks/worker-backlog.md` |
| DB CPU spike | RDS CPU > 80% | 10 min sustained | P2 | `runbooks/db-cpu.md` |
| Credential stuffing signal | Failed logins > 100/min | 1 min | P1 | `runbooks/credential-stuffing.md` |
| Health check failing | `/health` non-200 | 2 consecutive checks | P1 | `runbooks/health-check.md` |

---

## Dashboards

Maintained in CloudWatch Dashboards under the `Relay-Production` namespace:

- **API Health** — request rate, error rate, p50/p95/p99 latency, breakdown by endpoint.
- **Background Jobs** — queue depth, job throughput, failure rate per queue.
- **Database** — connection count, CPU utilization, slow query count.
- **WebSockets** — active connections, message throughput, connection error rate.
- **Auth** — signups per day, login success/failure rate, lockout events.
