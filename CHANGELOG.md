# Changelog

All notable spec and release changes are documented here.
Format: `[version] — YYYY-MM-DD`

---

## [0.5.0] — 2026-06-14

### Changed (breaking)
- **All specs** rewritten to the agent-first format: YAML front-matter (`owns`/`depends_on`), stable `INV-<AREA>-<n>` invariants, `Targets` tables, Given/When/Then acceptance criteria, and runnable `Verify` blocks. See ADR-005.
- `CONTRIBUTING.md` — replaced the prose spec template with the canonical agent-first template and ID conventions.
- `specs/ux/flows.md`, `specs/ux/copy.md` — converted from narrative to data tables governing UI behavior and string catalogs.

### Added
- `DECISIONS.md` — ADR-005: Agent-first spec format.

---

## [0.4.0] — 2024-11-01

### Added
- `specs/features/notifications.md` — full spec for in-app and email notifications
- `specs/api/websockets.md` — WebSocket event protocol

### Changed
- `specs/features/auth.md` — added Google SSO flow and account lockout policy
- `specs/data/schema.md` — added `notification_preferences` to User; added `notifications` table

---

## [0.3.0] — 2024-09-15

### Added
- `specs/features/payments.md` — billing and subscription spec
- `specs/data/retention.md` — data retention and deletion policy
- `specs/data/migrations.md` — migration policy and history

### Changed
- `specs/api/rest.md` — added `/billing` endpoint group

---

## [0.2.0] — 2024-08-01

### Added
- `specs/api/rest.md` — initial REST API contract
- `specs/api/errors.md` — error taxonomy and codes
- `specs/data/schema.md` — initial data model

---

## [0.1.0] — 2024-07-01

### Added
- `specs/overview.md` — initial product vision, goals, and glossary
- `specs/architecture.md` — initial system design
- `specs/features/auth.md` — email/password authentication spec
- `specs/ux/flows.md` — onboarding and task lifecycle flows
- `specs/ux/copy.md` — initial UI copy and tone guidelines
- `specs/ops/deployment.md`, `observability.md`, `security.md`
- `README.md`, `CONTRIBUTING.md`, `DECISIONS.md`
