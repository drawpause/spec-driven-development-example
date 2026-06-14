# REST API Specification

```yaml
spec: api/rest
status: active
last_updated: 2026-06-14
base_url: https://api.relay.app/v1
owns:
  - src/server/routes/**
  - openapi/relay.yaml
depends_on:
  - api/errors
  - features/auth
  - data/schema
```

## Summary

The REST contract for Relay: JSON over HTTPS, Bearer-token auth, and a generated OpenAPI document that is the machine-checkable source of truth for request/response shapes.

## Invariants

- **INV-REST-1:** Every endpoint and schema below MUST exist in `openapi/relay.yaml`; the served routes MUST validate requests and responses against it.
- **INV-REST-2:** All request and response bodies MUST be JSON; dates MUST be ISO-8601 UTC strings.
- **INV-REST-3:** Resource IDs in responses MUST be prefixed strings (`usr_`, `tsk_`, `prj_`, `wsp_`, `notif_`).
- **INV-REST-4:** List endpoints MUST return `{ "data": [...], "meta": { "total", "page", "per_page" } }`; `per_page` default 50, max 100.
- **INV-REST-5:** Errors MUST use the shape and codes in `api/errors`.
- **INV-REST-6:** Every route except `/auth/*` and `POST /webhooks/stripe` MUST require a valid access token (`ops/security` INV-SEC).
- **INV-REST-7:** `PATCH` endpoints MUST apply partial updates: only supplied fields change.
- **INV-REST-8:** `DELETE /tasks/:id` MUST be a soft delete (`deleted_at`); soft-deleted tasks MUST NOT appear in list endpoints.

## Contract

### Auth

| Method | Path | Request | Success | Errors |
|---|---|---|---|---|
| POST | `/auth/signup` | `{email,password,name}` | 201 `{access_token,user}` | `VALIDATION_ERROR`(422), `DUPLICATE_RESOURCE`(409) |
| POST | `/auth/login` | `{email,password}` | 200 `{access_token,user}` + `__relay_rt` cookie | `INVALID_CREDENTIALS`(401), `ACCOUNT_LOCKED`(423), `EMAIL_NOT_VERIFIED`(403) |
| POST | `/auth/refresh` | cookie `__relay_rt` | 200 `{access_token}` (rotates RT) | `TOKEN_INVALID`(401), `TOKEN_EXPIRED`(401) |
| POST | `/auth/logout` | — | 204 (clears cookie) | — |

`user` object: `{ id, email, name, email_verified }`.

### Tasks

`GET /projects/:projectId/tasks` — query: `status`, `assignee_id`, `due_before` (ISO), `page`, `per_page`.

```json
{
  "data": [{
    "id": "tsk_01", "title": "Build login page", "status": "in_progress",
    "assignee": { "id": "usr_01", "name": "Jane Doe" },
    "due_at": "2024-12-01T00:00:00Z",
    "created_at": "2024-11-01T10:00:00Z", "updated_at": "2024-11-02T08:30:00Z"
  }],
  "meta": { "total": 42, "page": 1, "per_page": 50 }
}
```

| Method | Path | Body | Success | Errors |
|---|---|---|---|---|
| POST | `/projects/:projectId/tasks` | `{title, assignee_id?, due_at?, status?}` | 201 task | `VALIDATION_ERROR`(422), `RESOURCE_NOT_FOUND`(404), `FORBIDDEN`(403) |
| PATCH | `/tasks/:taskId` | subset of `{title,status,assignee_id,due_at,description}` | 200 task | `VALIDATION_ERROR`(422), `RESOURCE_NOT_FOUND`(404) |
| DELETE | `/tasks/:taskId` | — | 204 (soft delete) | `RESOURCE_NOT_FOUND`(404) |

### Notifications

| Method | Path | Notes |
|---|---|---|
| GET | `/notifications` | query `read`(bool), `page`, `per_page`; `meta` adds `unread_count` |
| POST | `/notifications/read-all` | 204; sets `read_at` on all unread (notifications INV-NOTIF-6) |

### Billing (owner only — INV-CORE-7)

| Method | Path | Body | Success |
|---|---|---|---|
| GET | `/workspaces/:id/billing` | — | 200 `{plan,status,seats,current_period_end,payment_method}` |
| POST | `/workspaces/:id/billing/upgrade` | `{plan,seats}` | 200 billing; errors `FORBIDDEN`(403), `VALIDATION_ERROR`(422) |

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-REST-1 | `openapi/relay.yaml` + `src/server/middleware/validate.ts` | `validateAgainstOpenAPI` |
| INV-REST-4 | `src/server/lib/paginate.ts` | `paginate` |
| INV-REST-6 | `src/server/middleware/auth.ts` | `requireAccessToken` |
| INV-REST-7 | `src/server/routes/tasks.ts` | `patchTask` |
| INV-REST-8 | `src/server/routes/tasks.ts` | `deleteTask` |

## Acceptance

- **AC-REST-1** (INV-REST-1): GIVEN the running server WHEN the contract test suite replays every documented request THEN responses validate against `openapi/relay.yaml`.
- **AC-REST-2** (INV-REST-4): GIVEN `per_page=500` WHEN listing THEN the server clamps to 100.
- **AC-REST-3** (INV-REST-7): GIVEN a task WHEN `PATCH` sends only `status` THEN `title` and other fields are unchanged.
- **AC-REST-4** (INV-REST-8): GIVEN a deleted task WHEN listing its project THEN it is absent.
- **AC-REST-5** (INV-REST-6): GIVEN no token WHEN calling `GET /notifications` THEN response is 401.

## Verify

```bash
npm run test:contract        # INV-REST-1,2,3,4,5 against openapi/relay.yaml
npm test -- routes/tasks     # INV-REST-7,8
npx @redocly/cli lint openapi/relay.yaml
```
