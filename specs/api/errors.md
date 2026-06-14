# API Error Specification

```yaml
spec: api/errors
status: active
last_updated: 2026-06-14
owns:
  - src/server/errors/**
  - src/server/middleware/error-handler.ts
depends_on: []
```

## Summary

The single, stable error taxonomy every Relay endpoint uses: machine-readable `code`, human `message`, and an optional typed `details` object.

## Invariants

- **INV-ERR-1:** Every error response MUST have body `{ "error": { "code", "message", "details" } }`.
- **INV-ERR-2:** `code` MUST be one of the Error Codes table below and MUST be stable across API versions; clients branch on `code`, never on `message`.
- **INV-ERR-3:** Each `code` MUST map to exactly the HTTP status in the table.
- **INV-ERR-4:** `details` MUST match the documented shape for that code (below); absent details MUST be `{}`.
- **INV-ERR-5:** `500` responses MUST NOT leak internal detail (stack traces, SQL, secrets) in `message` or `details`.
- **INV-ERR-6:** Introducing a new error code MUST add a row here before it is returned by any route (enforced in review).

## Contract

### Shape
```json
{ "error": { "code": "RESOURCE_NOT_FOUND", "message": "The requested task does not exist.", "details": {} } }
```

### HTTP status semantics
| Status | Meaning |
|---|---|
| 400 | Malformed JSON or invalid parameter type |
| 401 | Unauthenticated — missing/invalid/expired token |
| 403 | Forbidden — authenticated but not authorized |
| 404 | Resource not found |
| 409 | Conflict — duplicate unique attribute |
| 422 | Validation error — parseable but failed business rules |
| 423 | Locked — account temporarily locked |
| 429 | Rate limited |
| 500 | Internal server error |

### Error Codes
| Code | Status | When |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | Email or password incorrect |
| `TOKEN_EXPIRED` | 401 | Access token expired; client should refresh |
| `TOKEN_INVALID` | 401 | Token malformed or signature invalid |
| `ACCOUNT_LOCKED` | 423 | Too many failed logins |
| `EMAIL_NOT_VERIFIED` | 403 | Email not yet verified |
| `FORBIDDEN` | 403 | Lacks required role/membership |
| `PLAN_LIMIT_EXCEEDED` | 403 | Action would exceed plan limits |
| `VALIDATION_ERROR` | 422 | One or more fields failed validation |
| `RESOURCE_NOT_FOUND` | 404 | Resource missing or not accessible |
| `DUPLICATE_RESOURCE` | 409 | Unique attribute already exists |
| `RATE_LIMITED` | 429 | Too many requests; see `Retry-After` |

### `details` shapes
```json
// ACCOUNT_LOCKED
{ "locked_until": "2024-11-01T14:15:00Z" }
// PLAN_LIMIT_EXCEEDED
{ "resource": "projects", "limit": 3, "current": 3, "plan": "free", "upgrade_to": "pro" }
// VALIDATION_ERROR
{ "fields": { "email": "Must be a valid email address.", "password": "Must be at least 10 characters." } }
```

### Client handling guide (normative)
```
TOKEN_EXPIRED        → call /auth/refresh, retry original request
ACCOUNT_LOCKED       → show lockout message with details.locked_until
PLAN_LIMIT_EXCEEDED  → show upgrade prompt
VALIDATION_ERROR     → render details.fields inline
500                  → generic message; do not auto-retry
```

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-ERR-1,3 | `src/server/middleware/error-handler.ts` | `errorHandler` |
| INV-ERR-2,4 | `src/server/errors/codes.ts` | `ERROR_CODES`, `DetailsByCode` |
| INV-ERR-5 | `src/server/middleware/error-handler.ts` | `sanitizeInternal` |

## Acceptance

- **AC-ERR-1** (INV-ERR-1,3): GIVEN any documented error path WHEN triggered THEN body matches the shape and HTTP status equals the table value.
- **AC-ERR-2** (INV-ERR-4): GIVEN `PLAN_LIMIT_EXCEEDED` THEN `details` contains `resource,limit,current,plan,upgrade_to`.
- **AC-ERR-3** (INV-ERR-5): GIVEN an unhandled exception THEN the 500 body contains no stack trace, SQL, or env value.
- **AC-ERR-4** (INV-ERR-2): GIVEN the codes module WHEN compared to this table THEN they are identical sets.

## Verify

```bash
npm test -- errors                 # INV-ERR-1..5
npm run check:error-codes          # AC-ERR-4: codes.ts ⇄ this spec table
```
