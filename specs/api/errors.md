# API Error Specification

**Last Updated:** 2024-11-01

---

## Error Shape

All API errors return a consistent JSON body:

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested task does not exist.",
    "details": {}
  }
}
```

- `code` — Machine-readable string constant. Stable across API versions. Clients should branch on this.
- `message` — Human-readable description. Do not parse or pattern-match on this; it may change.
- `details` — Optional object with context-specific data. Shape varies by error code (see below).

---

## HTTP Status Codes

| Status | Meaning |
|---|---|
| `400` | Bad request — malformed JSON or invalid parameter type |
| `401` | Unauthenticated — missing, invalid, or expired token |
| `403` | Forbidden — authenticated but not authorized for this action |
| `404` | Resource not found |
| `409` | Conflict — e.g., duplicate email on signup |
| `422` | Validation error — input was parseable but failed business validation |
| `423` | Locked — account is temporarily locked due to too many failed logins |
| `429` | Rate limited — too many requests |
| `500` | Internal server error — something unexpected went wrong on our end |

---

## Error Codes

### Authentication

| Code | Status | Description |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | Email or password is incorrect. |
| `TOKEN_EXPIRED` | 401 | The JWT access token has expired. Client should refresh. |
| `TOKEN_INVALID` | 401 | Token is malformed or the signature does not verify. |
| `ACCOUNT_LOCKED` | 423 | Too many consecutive failed login attempts. |
| `EMAIL_NOT_VERIFIED` | 403 | User has not verified their email address. |

**ACCOUNT_LOCKED details:**
```json
{ "details": { "locked_until": "2024-11-01T14:15:00Z" } }
```

### Authorization

| Code | Status | Description |
|---|---|---|
| `FORBIDDEN` | 403 | Authenticated user lacks the required role or membership. |
| `PLAN_LIMIT_EXCEEDED` | 403 | The action would exceed the workspace's plan limits. |

**PLAN_LIMIT_EXCEEDED details:**
```json
{
  "details": {
    "resource": "projects",
    "limit": 3,
    "current": 3,
    "plan": "free",
    "upgrade_to": "pro"
  }
}
```

### Validation

| Code | Status | Description |
|---|---|---|
| `VALIDATION_ERROR` | 422 | One or more fields failed validation. |

**VALIDATION_ERROR details:**
```json
{
  "details": {
    "fields": {
      "email": "Must be a valid email address.",
      "password": "Must be at least 10 characters."
    }
  }
}
```

### Resources

| Code | Status | Description |
|---|---|---|
| `RESOURCE_NOT_FOUND` | 404 | The requested resource does not exist or is not accessible to this user. |
| `DUPLICATE_RESOURCE` | 409 | A resource with those unique attributes already exists. |

### Rate Limiting

| Code | Status | Description |
|---|---|---|
| `RATE_LIMITED` | 429 | Too many requests. See `Retry-After` header for seconds to wait. |

---

## Client Handling Guide

```
if (error.code === "TOKEN_EXPIRED")  -> call /auth/refresh, retry original request
if (error.code === "ACCOUNT_LOCKED") -> show lockout message, display locked_until time
if (error.code === "PLAN_LIMIT_EXCEEDED") -> show upgrade prompt
if (error.code === "VALIDATION_ERROR") -> show field-level errors from details.fields
if (status === 500) -> show generic error, do not retry automatically
```
