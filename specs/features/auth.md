# Feature: Authentication

**Last Updated:** 2024-11-01
**Status:** Active

---

## Summary

Users authenticate with Relay using email + password or Google SSO. Sessions are maintained via short-lived JWTs and long-lived refresh tokens stored in httpOnly cookies.

---

## Goals
- Allow users to sign up, log in, and log out securely.
- Support Google SSO as a first-class alternative to email/password.
- Prevent unauthorized access to any workspace, project, or task.

## Non-Goals
- SAML / enterprise SSO (planned for v2).
- Magic link (passwordless email) login.
- Phone number or SMS authentication.

---

## Requirements

### Functional
- [ ] Users can sign up with email and password.
- [ ] Users can log in with email and password.
- [ ] Users can log in with Google OAuth 2.0.
- [ ] Users can log out, which invalidates their current refresh token.
- [ ] Users can reset their password via a time-limited email link.
- [ ] Access tokens expire after 15 minutes.
- [ ] Refresh tokens expire after 30 days, or immediately on logout.
- [ ] Refresh tokens are rotated on every use.

### Non-Functional
- Login must complete in < 500ms at p95.
- Passwords must be hashed with bcrypt, cost factor >= 12.
- After 10 consecutive failed logins, the account is locked for 15 minutes.

---

## Flows

### Email / Password Sign-Up

1. User submits email, password, and name.
2. System validates: email format valid; password >= 10 characters; email not already registered.
3. Password is hashed and stored. A verification email is sent with a 24-hour token.
4. User is logged in immediately (access token issued) but flagged `email_verified = false`.
5. Certain actions (e.g., inviting others) are blocked until email is verified.
6. User clicks the link in email -> account is marked `email_verified_at = now()`.

### Email / Password Login

1. User submits email and password.
2. System looks up user by email. If not found, return a generic error (do not reveal whether the email exists).
3. Password is compared against stored hash.
4. On match: issue access token and refresh token.
5. On failure: increment `failed_login_attempts`. If count reaches 10, set `locked_until = now() + 15 minutes`.

### Google SSO

1. User clicks "Continue with Google."
2. Frontend redirects to Google OAuth consent screen.
3. Google returns an authorization `code` to `GET /auth/google/callback`.
4. Server exchanges the code for a Google ID token and verifies it.
5. If the email matches an existing account: log in. If not: create a new account (pre-verified, no password).
6. Issue access token and refresh token.

### Password Reset

1. User submits their email at `/auth/forgot-password`.
2. If the email exists, a reset link is emailed (valid for 1 hour). If not, the response is identical (no enumeration).
3. User clicks the link and is prompted to choose a new password.
4. New password is saved; all existing refresh tokens for that user are immediately invalidated.

---

## Acceptance Criteria

- [ ] Given a valid email and password, a user can sign up and receives a verification email.
- [ ] Given an unverified account, invitation actions are blocked with error `EMAIL_NOT_VERIFIED`.
- [ ] Given 10 consecutive failed logins, subsequent login attempts return `423 Locked` with `locked_until` in the response.
- [ ] Given a valid Google OAuth flow, a new user is created, pre-verified, and logged in.
- [ ] Given a refresh token rotation, the old token is immediately invalidated and cannot be reused.
- [ ] Given a password reset, all previous refresh tokens for that user are invalidated.
