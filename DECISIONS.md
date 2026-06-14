# Architectural Decision Records

Significant technical and product decisions with their rationale. Append new ADRs; never edit existing ones (add a superseding ADR instead).

---

## ADR-001: PostgreSQL as primary database

**Date:** 2024-07-01
**Status:** Accepted

**Context:** We needed a relational database with strong consistency, JSONB for flexible metadata, and solid full-text search.

**Decision:** Use PostgreSQL 16.

**Alternatives considered:**
- MySQL — weaker JSONB support, less mature extension ecosystem
- MongoDB — no joins; unnecessary for our relational data model
- PlanetScale — limited to MySQL dialect; branching model adds complexity we don't need

**Consequences:** Migrations must be backwards-compatible. All schema changes must be specced in `specs/data/schema.md` and logged in `specs/data/migrations.md` before implementation begins.

---

## ADR-002: JWT + refresh token auth (stateless)

**Date:** 2024-07-01
**Status:** Accepted

**Context:** We're building a stateless API consumed by a web client and eventually mobile apps. Server-side sessions add operational overhead and require sticky routing or a shared session store.

**Decision:** Short-lived JWTs (15 min) + long-lived refresh tokens (30 days) stored in httpOnly cookies. Refresh tokens are stored hashed in the DB and rotated on every use.

**Alternatives considered:**
- Session cookies — stateful; requires sticky sessions or a Redis session store as a hard dependency
- Long-lived JWTs only — cannot be revoked without a blocklist, defeating the purpose

**Consequences:** Logout must invalidate the current refresh token server-side. Password reset must invalidate all existing refresh tokens for that user. See `specs/features/auth.md`.

---

## ADR-003: Redis for job queue and caching

**Date:** 2024-08-01
**Status:** Accepted

**Context:** Notifications, webhooks, and email delivery need a reliable background job queue with retry support. We also need a caching layer for expensive DB queries.

**Decision:** Redis (BullMQ for queues, ioredis for caching).

**Alternatives considered:**
- RabbitMQ — more operationally complex; overkill at current scale
- AWS SQS — no built-in delayed jobs on the free tier; adds AWS coupling to local dev
- In-process queue — not durable across process restarts

**Consequences:** Redis is now a critical-path dependency and must be included in the HA setup in production. ElastiCache with automatic failover is required.

---

## ADR-004: Stripe for payments

**Date:** 2024-09-01
**Status:** Accepted

**Context:** We need subscription billing with invoicing, proration, seat-based pricing, and trial periods, without building billing infrastructure ourselves.

**Decision:** Stripe Billing with webhook-driven state sync to our database.

**Alternatives considered:**
- Paddle — better EU VAT handling but a less flexible API; harder to migrate away from
- Manual invoicing — not scalable; too much operational overhead at launch
- Recurly — similar to Stripe but smaller ecosystem and higher cost at our scale

**Consequences:** Stripe is the source of truth for all subscription state. Our `subscriptions` table mirrors it and is updated via webhooks. Never update subscription state except in response to a verified Stripe webhook. See `specs/features/payments.md`.

---

## ADR-005: Agent-first spec format

**Date:** 2026-06-14
**Status:** Accepted

**Context:** Coding agents now do most implementation work in this repo. Prose specs written "for a new engineer" forced agents to infer which files a rule governs, what exactly must hold, and how to confirm conformance — producing inconsistent results and silent drift between spec and code.

**Decision:** Rewrite every spec to a machine-executable format: YAML front-matter declaring `owns`/`depends_on`, numbered stable `INV-<AREA>-<n>` invariants in MUST/MUST NOT language, a `Targets` table mapping each invariant to a file/symbol, Given/When/Then acceptance criteria that cite invariant IDs, and a runnable `Verify` block. The canonical template lives in `CONTRIBUTING.md`.

**Alternatives considered:**
- Keep prose specs, add an agent "style guide" — relies on inference; no enforceable structure
- JSON/YAML-only specs — machine-precise but unreadable for human review and PRs
- Gherkin/`.feature` files only — good for acceptance, weak for invariants, schemas, and file ownership

**Consequences:** Markdown stays human-reviewable while every rule is addressable and testable. `owns` globs are authoritative for which spec governs a file; conflicting globs are a spec bug. Acceptance criteria without a cited invariant, or invariants absent from a `Targets` table, fail review. UX flow/copy specs become data tables rather than narrative.
