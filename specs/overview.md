# Relay — Product Overview

```yaml
spec: overview
status: active
last_updated: 2026-06-14
owns:
  - specs/**          # global invariants and the domain model bind all specs
depends_on: []
```

## Summary

Relay is a B2B SaaS platform for team project and task management: workspaces contain projects, projects contain tasks, tasks are assigned to members and update in real time.

## Domain Model

```
Workspace 1───* Project 1───* Task *───1 Assignee(User)
Workspace 1───* Membership *───1 User
Workspace 1───1 Subscription
User 1───* Notification
```

Canonical entity IDs are prefixed strings (see `data/schema`): `wsp_`, `prj_`, `tsk_`, `usr_`, `notif_`, `sub_`, `rt_`.

## Invariants

- **INV-CORE-1:** Every Project MUST belong to exactly one Workspace; every Task MUST belong to exactly one Project. No orphan tasks or projects.
- **INV-CORE-2:** A Task MUST have at most one assignee, and that assignee MUST be a member of the task's workspace.
- **INV-CORE-3:** Task `status` MUST be one of `todo`, `in_progress`, `in_review`, `done`, `cancelled`. No other value is writable.
- **INV-CORE-4:** Every workspace MUST have exactly one `owner` (role) at all times.
- **INV-CORE-5:** Authorization is workspace-scoped: a user MUST be a member of a workspace to read or write any of its projects or tasks (enforced by `features/auth` and `ops/security`).
- **INV-CORE-6:** p95 latency for any interactive API request MUST be < 300ms (measured per `ops/observability`).
- **INV-CORE-7:** Role capabilities MUST follow the Roles table below; a lower role MUST NOT gain a higher role's capability through any endpoint.

## Roles

| Role | Manage members | Project settings | Create/assign tasks | Billing | Read tasks |
|---|---|---|---|---|---|
| `owner` | yes | yes | yes | yes | yes |
| `admin` | yes | yes | yes | no | yes |
| `member` | no | no | yes (in own projects) | no | yes |
| `guest` | no | no | no | no | invited projects only |

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-CORE-1 | `src/db/schema.ts` | foreign keys `projects.workspace_id`, `tasks.project_id` |
| INV-CORE-2 | `src/tasks/assign.ts` | `assignTask` |
| INV-CORE-3 | `src/tasks/status.ts` | `TASK_STATUS` enum, `setStatus` |
| INV-CORE-4 | `src/workspaces/roles.ts` | `transferOwnership` |
| INV-CORE-5 | `src/auth/authorize.ts` | `requireWorkspaceMember` |
| INV-CORE-7 | `src/auth/authorize.ts` | `ROLE_CAPABILITIES` |

## Acceptance

- **AC-CORE-1** (INV-CORE-2): GIVEN user U not a member of workspace W WHEN assigning a task in W to U THEN the API returns `VALIDATION_ERROR` and no write occurs.
- **AC-CORE-2** (INV-CORE-3): GIVEN any task WHEN PATCH sets `status` to a value outside the enum THEN the API returns `VALIDATION_ERROR`.
- **AC-CORE-3** (INV-CORE-4): GIVEN a workspace with owner O WHEN O is removed without transfer THEN the operation is rejected; ownership transfer is required first.
- **AC-CORE-4** (INV-CORE-7): GIVEN a `member` WHEN calling a billing endpoint THEN the API returns `FORBIDDEN`.

## Non-Goals

Out of scope in v1 (do not implement): document editor, chat, enterprises > 500 seats, native mobile apps.

## Glossary

| Term | Definition |
|---|---|
| Workspace | Top-level org container; one billing subscription. |
| Project | Named collection of tasks within a workspace. |
| Task | Atomic unit of work: title, status, assignee, due date, comments. |
| Status | Task state: `todo`, `in_progress`, `in_review`, `done`, `cancelled`. |
| Assignee | The single member responsible for a task. |
| Notification | Alert delivered to a user on a relevant event. |

## Verify

```bash
npm test -- core/invariants     # INV-CORE-1..4,7
npm run lint:specs              # front-matter, owns globs, invariant/AC cross-refs
npm run perf:p95 -- --max 300   # INV-CORE-6 against staging traffic replay
```
