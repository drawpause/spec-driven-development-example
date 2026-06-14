# Relay

This is an example spec-driven-development spec.
Relay is a B2B SaaS platform for team project and task management. It helps teams organize work, track progress, and collaborate in real time.

## Quick Start

1. Clone the repository
2. Read [specs/overview.md](specs/overview.md) to understand the product
3. Read [specs/architecture.md](specs/architecture.md) for system design
4. Run `docker compose up` to start the dev environment

## Documentation

| Document | Purpose |
|---|---|
| [specs/overview.md](specs/overview.md) | Vision, goals, and glossary |
| [specs/architecture.md](specs/architecture.md) | System architecture |
| [specs/features/](specs/features/) | Per-feature behavioral specs |
| [specs/api/](specs/api/) | API contracts |
| [specs/data/](specs/data/) | Data models and lifecycle |
| [specs/ux/](specs/ux/) | User flows and UI copy |
| [specs/ops/](specs/ops/) | Deployment, observability, security |
| [DECISIONS.md](DECISIONS.md) | Architectural decision records |
| [CHANGELOG.md](CHANGELOG.md) | Spec and release history |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to write and update specs |

## Tech Stack

- **Backend:** Node.js + TypeScript, PostgreSQL, Redis
- **Frontend:** React, Tailwind CSS
- **Infrastructure:** AWS (ECS, RDS, ElastiCache), Terraform
- **Auth:** JWT + refresh tokens, Google SSO



