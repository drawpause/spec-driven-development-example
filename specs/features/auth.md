# Feature: Authentication

```yaml
spec: features/auth
status: active
last_updated: 2026-06-14
owns:
  - src/auth/**
  - src/server/routes/auth.ts
depends_on:
  - api/rest
  - api/errors
  - data/schema
  - ops/security
```

## Summary

Users authenticate with email + password or Google SSO; sessions use short-lived JWT access tokens plus rotating refresh tokens in an httpOnly cookie.

## Invariants

- **INV-AUTH-1:** Access tokens (JWT, HS256) MUST expire ≤ 900s (15 min) after issuance.
- **INV-AUTH-2:** Refresh tokens MUST expire ≤ 30 days after issuance and MUST be stored only as a bcrypt hash (`refresh_tokens.token_hash`), never in plaintext.
- **INV-AUTH-3:** Each refresh token MUST be single-use: on `/auth/refresh` the presented token is revoked and a new one issued (rotation). A revoked token MUST NOT authenticate.
- **INV-AUTH-4:** Passwords MUST be hashed with bcrypt, cost ≥ 12, and MUST be ≥ 10 characters at signup.
- **INV-AUTH-5:** After 10 consecutive failed logins, the account MUST be locked for 15 minutes (`locked_until = now()+15m`); login during lockout MUST return `ACCOUNT_LOCKED` (423).
- **INV-AUTH-6:** Login and password-reset responses MUST NOT reveal whether an email exists (no user enumeration).
- **INV-AUTH-7:** Logout MUST revoke the current refresh token; password reset MUST revoke ALL refresh tokens for that user.
- **INV-AUTH-8:** A user with `email_verified_at IS NULL` MUST be blocked from gating actions (e.g. inviting members) with `EMAIL_NOT_VERIFIED` (403).
- **INV-AUTH-9:** Google SSO login MUST verify the Google ID token signature and audience before trusting any claim; a new SSO account is created pre-verified with no password.
- **INV-AUTH-10:** Login MUST complete in < 500ms at p95.

## Contract

State transitions for an account record (`users`):

```
signup ─▶ email_verified_at = NULL ──(click link ≤24h)──▶ email_verified_at = now()
login_fail × 10 ─▶ locked_until = now()+15m ──(15m elapse OR successful reset)──▶ unlocked
```

Endpoints (full schemas in `api/rest`): `POST /auth/signup`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `POST /auth/forgot-password`, `GET /auth/google/callback`.

Token/cookie rules:
- Access token returned in body; refresh token set as `__relay_rt` httpOnly, `Secure`, `SameSite=Lax` cookie.
- `failed_login_attempts` resets to 0 on any successful login.

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-AUTH-1 | `src/auth/tokens.ts` | `issueAccessToken` (`expiresIn: 900`) |
| INV-AUTH-2,3,7 | `src/auth/refresh.ts` | `rotateRefreshToken`, `revokeRefreshToken`, `revokeAllForUser` |
| INV-AUTH-4 | `src/auth/password.ts` | `hashPassword`, `PASSWORD_MIN_LEN` |
| INV-AUTH-5 | `src/auth/lockout.ts` | `recordFailedLogin`, `assertNotLocked` |
| INV-AUTH-6 | `src/server/routes/auth.ts` | `login`, `forgotPassword` |
| INV-AUTH-8 | `src/auth/authorize.ts` | `requireVerifiedEmail` |
| INV-AUTH-9 | `src/auth/google.ts` | `verifyGoogleIdToken` |

## Acceptance

- **AC-AUTH-1** (INV-AUTH-1): GIVEN a fresh access token WHEN decoded THEN `exp - iat == 900`.
- **AC-AUTH-2** (INV-AUTH-3): GIVEN refresh token R WHEN `/auth/refresh` is called twice with R THEN the first returns 200 and the second returns `TOKEN_INVALID` (401).
- **AC-AUTH-3** (INV-AUTH-5): GIVEN 10 consecutive failed logins WHEN logging in again THEN response is 423 `ACCOUNT_LOCKED` with `details.locked_until`.
- **AC-AUTH-4** (INV-AUTH-6): GIVEN a non-existent email WHEN logging in THEN the error is identical to a wrong-password error for an existing email.
- **AC-AUTH-5** (INV-AUTH-7): GIVEN a password reset completes WHEN any pre-existing refresh token is presented THEN it is rejected.
- **AC-AUTH-6** (INV-AUTH-9): GIVEN a valid Google OAuth callback for a new email THEN an account is created with `email_verified_at` set and `password_hash = NULL`, and tokens are issued.
- **AC-AUTH-7** (INV-AUTH-8): GIVEN an unverified user WHEN inviting a member THEN response is 403 `EMAIL_NOT_VERIFIED`.

## Verify

```bash
npm test -- auth                       # INV-AUTH-1..9 unit/integration
npm run test:e2e -- auth-lockout       # AC-AUTH-3
npm run bench:login -- --p95-max 500   # INV-AUTH-10
```
