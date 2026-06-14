# REST API Specification

**Last Updated:** 2024-11-01
**Base URL:** `https://api.relay.app/v1`
**Auth:** Bearer token (JWT) in `Authorization: Bearer <token>` header.

---

## Conventions

- All request and response bodies are JSON.
- Dates are ISO 8601 UTC strings (e.g., `2024-11-01T14:30:00Z`).
- Paginated list endpoints return `{ "data": [...], "meta": { "total", "page", "per_page" } }`.
- Errors follow the shape defined in [errors.md](errors.md).
- Resource IDs are prefixed strings (e.g., `usr_01`, `tsk_42`).

---

## Auth

### POST /auth/signup
Register a new user.

**Request body:**
```json
{
  "email": "jane@example.com",
  "password": "supersecret123",
  "name": "Jane Doe"
}
```

**Response 201:**
```json
{
  "access_token": "eyJ...",
  "user": {
    "id": "usr_01",
    "email": "jane@example.com",
    "name": "Jane Doe",
    "email_verified": false
  }
}
```

**Errors:** `VALIDATION_ERROR` (422), `DUPLICATE_RESOURCE` (409 if email taken)

---

### POST /auth/login
Authenticate with email and password. Sets `__relay_rt` httpOnly cookie.

**Request body:** `{ "email": "...", "password": "..." }`

**Response 200:** Same shape as signup response.

**Errors:** `INVALID_CREDENTIALS` (401), `ACCOUNT_LOCKED` (423), `EMAIL_NOT_VERIFIED` (403)

---

### POST /auth/refresh
Exchange the `__relay_rt` cookie for a new access token. Also rotates the refresh token.

**Response 200:** `{ "access_token": "eyJ..." }`

**Errors:** `TOKEN_INVALID` (401), `TOKEN_EXPIRED` (401)

---

### POST /auth/logout
Invalidates the current refresh token. Clears the `__relay_rt` cookie.

**Response 204:** No content.

---

## Tasks

### GET /projects/:projectId/tasks
List tasks in a project.

**Query params:**
| Param | Type | Description |
|---|---|---|
| `status` | string | Filter by status (`todo`, `in_progress`, etc.) |
| `assignee_id` | string | Filter by assignee |
| `due_before` | ISO date | Filter tasks due before this date |
| `page` | int | Page number (default: 1) |
| `per_page` | int | Results per page (default: 50, max: 100) |

**Response 200:**
```json
{
  "data": [
    {
      "id": "tsk_01",
      "title": "Build login page",
      "status": "in_progress",
      "assignee": { "id": "usr_01", "name": "Jane Doe" },
      "due_at": "2024-12-01T00:00:00Z",
      "created_at": "2024-11-01T10:00:00Z",
      "updated_at": "2024-11-02T08:30:00Z"
    }
  ],
  "meta": { "total": 42, "page": 1, "per_page": 50 }
}
```

---

### POST /projects/:projectId/tasks
Create a new task.

**Request body:**
```json
{
  "title": "Build login page",
  "assignee_id": "usr_01",
  "due_at": "2024-12-01T00:00:00Z",
  "status": "todo"
}
```

**Response 201:** Full task object.

**Errors:** `VALIDATION_ERROR` (422), `RESOURCE_NOT_FOUND` (404 if project not found), `FORBIDDEN` (403)

---

### PATCH /tasks/:taskId
Partially update a task. Include only the fields to change.

**Request body:** Any subset of: `title`, `status`, `assignee_id`, `due_at`, `description`.

**Response 200:** Updated task object.

---

### DELETE /tasks/:taskId
Soft-delete a task (sets `deleted_at`). Does not appear in list endpoints after deletion.

**Response 204:** No content.

---

## Notifications

### GET /notifications
List the authenticated user's notifications, newest first.

**Query params:** `read` (boolean, filter by read status), `page`, `per_page`.

**Response 200:**
```json
{
  "data": [
    {
      "id": "notif_09",
      "event": "task_assigned",
      "payload": { "task_id": "tsk_05", "task_title": "Review PR #42" },
      "read_at": null,
      "created_at": "2024-11-01T14:00:00Z"
    }
  ],
  "meta": { "total": 7, "unread_count": 3, "page": 1, "per_page": 50 }
}
```

### POST /notifications/read-all
Mark all unread notifications as read for the authenticated user.

**Response 204:** No content.

---

## Billing

### GET /workspaces/:workspaceId/billing
Get the current subscription and payment method. Owner only.

**Response 200:**
```json
{
  "plan": "pro",
  "status": "active",
  "seats": 8,
  "current_period_end": "2024-12-01T00:00:00Z",
  "payment_method": { "brand": "visa", "last4": "4242" }
}
```

### POST /workspaces/:workspaceId/billing/upgrade
Change the subscription plan or seat count.

**Request body:** `{ "plan": "pro", "seats": 12 }`

**Response 200:** Updated billing object.

**Errors:** `FORBIDDEN` (403 if not owner), `VALIDATION_ERROR` (422)
