# Security

**Last Updated:** 2024-11-01

---

## Threat Model

We focus on the following primary threat categories:

1. **Credential theft** — phishing, password reuse attacks, brute force.
2. **Unauthorized access** — accessing another user's workspace, project, or task data.
3. **Data exfiltration** — bulk extraction of user data via the API.
4. **Injection attacks** — SQL injection, cross-site scripting (XSS).
5. **Payment fraud** — abuse of billing endpoints or Stripe webhook forgery.
6. **Dependency vulnerabilities** — compromised or vulnerable npm packages.

---

## Authentication Security

- **Passwords:** bcrypt with cost factor >= 12. Never stored in plaintext.
- **JWT:** HS256, 15-minute expiry. Secret rotated every 6 months.
- **Refresh tokens:** Stored as a bcrypt hash in the DB. Rotated on every use. Revoked on logout, password reset, and account deletion.
- **Account lockout:** 10 consecutive failed login attempts locks the account for 15 minutes. Owner receives an email notification on lockout.

---

## Authorization

- All API routes require a valid JWT except `/auth/*` and `POST /webhooks/stripe`.
- Authorization is enforced at the resource level: a user must be a member of a workspace to access any of its projects or tasks.
- Workspace `owner` role is required to access all billing endpoints.
- Admins cannot access billing; Guests are read-only.
- There is no internal "superuser" bypass. Engineers access production data only through approved read-only tooling with audit logging.

---

## API Security

**Rate limiting:**
- `/auth/signup`, `/auth/login`, `/auth/forgot-password`: 10 requests/minute per IP.
- All other routes: 1000 requests/minute per authenticated user.
- Rate limit responses include `Retry-After` header.

**Input handling:**
- All inputs validated and sanitized at the API layer before touching the database.
- All SQL queries use parameterized statements. String interpolation in SQL is forbidden and enforced by a linting rule.
- File uploads: validated for MIME type and size (max 25 MB) before writing to S3. Filenames are replaced with random IDs.

**Transport and headers:**
- TLS 1.2+ required. HSTS enforced (`max-age=31536000; includeSubDomains`).
- CORS: `Access-Control-Allow-Origin` set to `https://relay.app` only. No wildcard.
- Security headers on all responses: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`.

**Webhook security:**
- All Stripe webhook requests are verified using the `Stripe-Signature` header before processing.

---

## Secrets Management

- All secrets live in AWS Secrets Manager. Never in `.env` files, code, or environment variable configs committed to git.
- `.env` is in `.gitignore`. A pre-commit hook blocks any file containing the string `sk_live_` or `SECRET`.
- Secret rotation schedule: `JWT_SECRET` every 6 months; database passwords every 90 days.
- All AWS Secrets Manager access is logged in CloudTrail.

---

## Dependency Management

- `npm audit` runs on every PR. PRs with high or critical severity findings are blocked from merging.
- Dependabot is enabled for automatic patch-version security updates.
- Major dependency updates are reviewed manually before merging.

---

## Incident Response

1. **Detect:** Alert fires in PagerDuty or a report arrives at `security@relay.app`.
2. **Contain:** On-call engineer revokes affected tokens, blocks IPs, or disables endpoints as appropriate.
3. **Assess:** Determine scope — which users and data were affected, for how long.
4. **Notify:** If personal data was involved, notify affected users within 72 hours (GDPR Article 33 requirement).
5. **Post-mortem:** Written post-mortem published internally within 7 days. Includes root cause, timeline, and follow-up actions with owners and due dates.

**Responsible disclosure:** Report vulnerabilities to `security@relay.app`. Acknowledge within 48 hours. We do not pursue legal action against good-faith researchers.
