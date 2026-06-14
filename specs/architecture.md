# System Architecture

**Last Updated:** 2024-11-01
**Status:** Active

---

## Overview

Relay is a monolithic API-first application deployed as containerized services on AWS ECS. The frontend is a React SPA served from S3 + CloudFront.

```
+--------------------------------------------------+
|                     Clients                      |
|          Web (React)        Mobile (future)      |
+--------------------+-----------------------------+
                     | HTTPS / WSS
+--------------------v-----------------------------+
|               API Server (Node.js)               |
|           REST API + WebSocket Gateway           |
+--------+-------------------+----------+----------+
         |                   |          |
+--------v-------+  +--------v----+  +--v---------+
|  PostgreSQL    |  |    Redis    |  |     S3     |
|  (Primary DB)  |  | Queue/Cache |  | (Uploads)  |
+----------------+  +-------------+  +------------+
         |
+--------v-----------------------------------------+
|              Background Workers                  |
|   Email (SES)  |  Webhooks  |  Notifications    |
+--------------------------------------------------+
```

---

## Components

### API Server
- Node.js 20 + TypeScript
- Express for HTTP routing
- `ws` library for WebSocket connections
- Runs on ECS Fargate; auto-scales 2-10 instances based on CPU and request queue depth

### Database
- PostgreSQL 16 on RDS (Multi-AZ in production)
- Migrations managed by `node-pg-migrate`
- Read replica for analytics and reporting queries

### Cache and Queue
- Redis 7 on ElastiCache (cluster mode off, single primary + replica)
- BullMQ for background jobs (email delivery, webhooks, notification fan-out)
- LRU cache for workspace settings and feature flags (5-minute TTL)

### File Storage
- S3 for task attachments and user avatars
- Presigned URLs for direct browser uploads (max 25 MB per file, 10 min expiry)
- CloudFront CDN for file delivery

### Frontend
- React 18 + TypeScript
- Tailwind CSS
- Deployed to S3 + CloudFront
- Communicates with the API over REST (mutations and initial loads) and WebSocket (real-time updates)

---

## Key Design Principles

- The API is the single integration surface. The frontend is a first-party client, not a privileged one.
- All mutations go through the API; the frontend never writes to the DB directly.
- Background jobs are the only place external HTTP calls are made (email, Stripe webhooks). Never in the request path.

For rationale on specific choices, see [DECISIONS.md](../DECISIONS.md).

---

## Environments

| Environment | Purpose | Database | Notes |
|---|---|---|---|
| `local` | Developer machines | Docker Postgres | Seeded with fixture data |
| `staging` | Integration testing | RDS single-AZ | Mirrors production config; reset weekly |
| `production` | Live traffic | RDS Multi-AZ | Read replica enabled |
