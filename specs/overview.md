# Relay — Product Overview

**Last Updated:** 2024-11-01
**Status:** Active

---

## Vision

Relay gives teams a single place to plan, assign, and track work — without the overhead of tools built for enterprise complexity.

## Problem

Small and mid-size teams bounce between spreadsheets, chat threads, and heavyweight project management tools that require weeks to configure. Work gets lost in the gaps between those tools.

## Solution

Relay is a lightweight project management platform with just enough structure: workspaces, projects, tasks, assignees, and real-time updates. Nothing more by default; everything extendable via API.

---

## Goals

- Help teams of 2–200 people ship work without friction.
- Provide a clean API so engineering teams can build automations and integrations.
- Be fast: every interaction < 300ms at p95.
- Reach product-market fit with a < 3 minute time-to-first-task for new users.

## Non-Goals

- We are **not** a document editor (no Notion/Google Docs replacement in v1).
- We are **not** a chat tool (no Slack replacement).
- We are **not** targeting enterprises > 500 seats in v1.
- We are **not** building a mobile app in v1.

---

## User Roles

| Role | Description |
|---|---|
| **Owner** | Creates and manages a workspace. Responsible for billing. Can do everything Admins can. |
| **Admin** | Manages members and project settings within a workspace. Cannot access billing. |
| **Member** | Creates, assigns, and completes tasks within projects they belong to. |
| **Guest** | View-only access to specific projects they have been explicitly invited to. |

---

## Glossary

| Term | Definition |
|---|---|
| **Workspace** | The top-level container for an organization. Tied to one billing subscription. |
| **Project** | A named collection of tasks within a workspace. Has an optional description and due date. |
| **Task** | The atomic unit of work. Has a title, status, assignee, due date, and comment thread. |
| **Status** | A task's current state: `todo`, `in_progress`, `in_review`, `done`, `cancelled`. |
| **Assignee** | The single member responsible for completing a task. |
| **Comment** | A message in a task's thread. Supports @mentions and file attachments. |
| **Notification** | An alert delivered to a user when a relevant event occurs (assignment, mention, etc.). |

---

## Success Metrics

- Time to first task created: < 3 minutes from signup.
- Weekly active users / total users > 60%.
- Net Promoter Score > 40 within 6 months of launch.
- API error rate < 0.1% over any 24-hour window.
