# Security

```yaml
spec: ops/security
status: active
last_updated: 2026-06-14
owns:
  - src/server/middleware/security.ts
  - src/server/middleware/rate-limit.ts
  - .github/workflows/security.yml
depends_on:
  - features/auth
  - features/payments
  - overview
```

## Summary

Relay's enforceable security controls: authentication hardening, resource-level authorization, API hardening, secrets handling, dependency gates, and incident response.

## Threat Model

Primary categories: (1) credential theft, (2) unauthorized cross-tenant access, (3) bulk data exfiltration, (4) injection (SQLi/XSS), (5) payment fraud / webhook forgery, (6) dependency vulnerabilities.

## Invariants

- **INV-SEC-1:** Every route except `/auth/*` and `POST /webhooks/stripe` MUST require a valid access token (REST INV-REST-6).
- **INV-SEC-2:** Authorization MUST be resource-scoped: a user MUST be a member of a workspace to access any project/task within it; there MUST be no superuser bypass.
- **INV-SEC-3:** Billing routes MUST require workspace `owner` (overview INV-CORE-7).
- **INV-SEC-4:** Passwords MUST use bcrypt cost ≥ 12; JWTs HS256 with 15-min expiry; `JWT_SECRET` rotated ≤ 6 months; refresh tokens hashed and rotated (auth INV-AUTH-1..4).
- **INV-SEC-5:** Rate limits MUST be enforced: `/auth/signup|login|forgot-password` 10/min/IP; all other routes 1000/min/user; responses include `Retry-After`.
- **INV-SEC-6:** All SQL MUST use parameterized statements; string interpolation into SQL MUST be blocked by lint.
- **INV-SEC-7:** Uploads MUST be validated for MIME type and ≤ 25 MB before S3 write; stored filenames MUST be random IDs.
- **INV-SEC-8:** Responses MUST set HSTS, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, and a CSP; TLS 1.2+ only.
- **INV-SEC-9:** CORS `Access-Control-Allow-Origin` MUST be `https://relay.app` exactly; no wildcard.
- **INV-SEC-10:** Stripe webhooks MUST be signature-verified before processing (payments INV-PAY-2).
- **INV-SEC-11:** Secrets MUST live only in AWS Secrets Manager; a pre-commit hook MUST block files containing `sk_live_` or `SECRET`.
- **INV-SEC-12:** PRs with high/critical `npm audit` findings MUST be blocked from merge.
- **INV-SEC-13:** If personal data is involved in an incident, affected users MUST be notified within 72 hours (GDPR Art. 33).

## Contract

### Rate limits
| Scope | Limit |
|---|---|
| `/auth/signup`, `/auth/login`, `/auth/forgot-password` | 10/min per IP |
| all other authenticated routes | 1000/min per user |

### Response headers (all responses)
`Strict-Transport-Security: max-age=31536000; includeSubDomains`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `Content-Security-Policy: <policy>`.

### Secret rotation
`JWT_SECRET` every 6 months; database passwords every 90 days; all Secrets Manager access logged in CloudTrail.

### Incident response
`detect → contain (revoke tokens / block IPs / disable endpoints) → assess scope → notify (≤72h if PII) → post-mortem (≤7 days, root cause + owners + due dates)`. Responsible disclosure: `security@relay.app`, acknowledged ≤ 48h.

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-SEC-1,2,3 | `src/auth/authorize.ts` | `requireAccessToken`, `requireWorkspaceMember`, `requireOwner` |
| INV-SEC-5 | `src/server/middleware/rate-limit.ts` | `RATE_LIMITS` |
| INV-SEC-6 | `.eslintrc` + `src/db/sql.ts` | `no-raw-sql` rule, `sql` tagged template |
| INV-SEC-7 | `src/server/uploads.ts` | `validateUpload` |
| INV-SEC-8,9 | `src/server/middleware/security.ts` | `securityHeaders`, `corsOptions` |
| INV-SEC-11 | `.husky/pre-commit` | secret scan |
| INV-SEC-12 | `.github/workflows/security.yml` | `audit` job |

## Acceptance

- **AC-SEC-1** (INV-SEC-2): GIVEN user not in workspace W WHEN requesting a task in W THEN response is 404/403 with no data leak.
- **AC-SEC-2** (INV-SEC-5): GIVEN 11 logins in a minute from one IP THEN the 11th returns 429 with `Retry-After`.
- **AC-SEC-3** (INV-SEC-8,9): GIVEN any response WHEN headers inspected THEN all required headers are present and CORS origin is exactly `https://relay.app`.
- **AC-SEC-4** (INV-SEC-6): GIVEN code interpolating a variable into SQL THEN ESLint fails.
- **AC-SEC-5** (INV-SEC-11): GIVEN a commit containing `sk_live_…` THEN the pre-commit hook blocks it.
- **AC-SEC-6** (INV-SEC-7): GIVEN a 30 MB upload THEN it is rejected before any S3 write.

## Verify

```bash
npm test -- security/headers     # INV-SEC-8,9
npm test -- security/authz       # INV-SEC-1,2,3
npm test -- security/rate-limit  # INV-SEC-5
npm run lint                     # INV-SEC-6 (no-raw-sql)
npm audit --audit-level=high     # INV-SEC-12
```
