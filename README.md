# Relay

An example of **spec-driven development for coding agents**.

Relay is a B2B SaaS platform for team project and task management. This repository
contains its specs only — but the specs are written so that a coding agent can
implement, modify, and verify the product directly from them. Each spec declares the
files it governs (`owns`), the rules that must always hold (`Invariants`), where to
implement them (`Targets`), and how to prove conformance (`Verify`).

## How an Agent Uses These Specs

1. Read [specs/overview.md](specs/overview.md) for the domain model and global invariants.
2. Read [specs/architecture.md](specs/architecture.md) for system boundaries and where code lives.
3. To implement or change a behavior, open the spec whose `owns` globs match the target files.
4. Implement each invariant at its mapped `Targets` file/symbol.
5. Run the spec's `Verify` commands; they exit non-zero on nonconformance.

A spec is the contract. If code and spec disagree, the spec wins (see [CONTRIBUTING.md](CONTRIBUTING.md)).

## Spec Map

| Document | Spec id | Purpose |
|---|---|---|
| [specs/overview.md](specs/overview.md) | `overview` | Domain model, roles, global invariants, glossary |
| [specs/architecture.md](specs/architecture.md) | `architecture` | System boundaries, components, code layout |
| [specs/features/](specs/features/) | `features/*` | Per-feature behavioral contracts |
| [specs/api/](specs/api/) | `api/*` | REST + WebSocket contracts, error taxonomy |
| [specs/data/](specs/data/) | `data/*` | Schema, migrations, retention |
| [specs/ux/](specs/ux/) | `ux/*` | Flows and copy (as data tables, not prose) |
| [specs/ops/](specs/ops/) | `ops/*` | Deployment, observability, security |
| [DECISIONS.md](DECISIONS.md) | — | Architectural decision records (the "why") |
| [CHANGELOG.md](CHANGELOG.md) | — | Spec and release history |
| [CONTRIBUTING.md](CONTRIBUTING.md) | — | Canonical spec template + ID conventions |

## Conventions (agent-facing)

- Invariants are `INV-<AREA>-<n>`; acceptance criteria `AC-<AREA>-<n>` cite the invariants they prove.
- Every spec front-matter lists `owns` (authoritative file globs) and `depends_on` (other specs).
- `Verify` blocks are runnable headless and are the definition of "done."

## Tech Stack

- **Backend:** Node.js 20 + TypeScript, PostgreSQL 16, Redis 7
- **Frontend:** React 18, Tailwind CSS
- **Infrastructure:** AWS (ECS, RDS, ElastiCache), Terraform
- **Auth:** JWT + refresh tokens, Google SSO
