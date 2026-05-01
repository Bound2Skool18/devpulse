# DevPulse — Technical Design Document

**Author:** Your Name  
**Created:** 2026-05-01  
**Last Updated:** 2026-05-01  
**Status:** Draft  
**Reviewers:** Self-review

---

## 📑 Table of Contents

1. [Problem Statement](#problem-statement)
2. [Goals & Non-Goals](#goals--non-goals)
3. [User Personas](#user-personas)
4. [System Architecture](#system-architecture)
5. [Database Schema](#database-schema)
6. [API Endpoints](#api-endpoints)
7. [Tech Stack Justification](#tech-stack-justification)
8. [Key Design Decisions](#key-design-decisions)
9. [Risks & Open Questions](#risks--open-questions)
10. [Milestones](#milestones)


## Problem Statement

Engineering teams rely on a fragmented stack of tools to coordinate daily work — 
Slack for standups, Jira for tickets, GitHub for code reviews, PagerDuty for 
on-call, and Notion for docs. Context switching between these tools wastes 
time and causes information to get lost between platforms.

DevPulse consolidates the four most common engineering coordination workflows 
into a single application:

1. **Async standups** — replacing daily Slack ceremonies
2. **Ticket tracking** — a lightweight Kanban board
3. **PR visibility** — surfacing GitHub PR activity in one place
4. **On-call awareness** — knowing who's responsible right now

The goal is not to replace specialized tools (Jira will always be more powerful 
than our Kanban board), but to give small teams a single, unified hub for the 
80% of coordination work they do every day.


## Goals & Non-Goals

### ✅ Goals (In Scope)

- Authenticate users with email/password and JWT tokens
- Support three roles: Engineer, Tech Lead, Manager
- Allow users to submit and view daily standups
- Provide a drag-and-drop Kanban board for tickets
- Integrate with GitHub to display open PRs
- Send real-time notifications via WebSockets
- Display team productivity metrics on a dashboard
- Manage on-call rotation schedules
- Deploy to production with CI/CD

### ❌ Non-Goals (Out of Scope)

- Mobile native apps (web-only for v1)
- Multi-tenant SaaS architecture (single team only)
- Slack/email notification delivery (in-app only for v1)
- Time tracking or billing features
- Replacing full Jira functionality (no epics, sprints, or burndown charts)
- AI-powered features (potential Week 8 stretch goal)
- Internationalization / multi-language support


## User Personas

### 👩‍💻 Persona 1: Engineer ("Eva")
- **Role:** Mid-level software engineer
- **Daily needs:** Submit standup, check assigned tickets, see PR review requests, get notified of blockers
- **Pain points:** Forgetting to post standup, losing track of which PR needs review

### 👨‍💼 Persona 2: Tech Lead ("Theo")
- **Role:** Senior engineer leading a team of 4–6
- **Daily needs:** Read team standups, assign tickets, monitor team velocity, identify blockers
- **Pain points:** No single view of team activity, missing context on what people are working on

### 👩‍💼 Persona 3: Manager ("Maria")
- **Role:** Engineering manager overseeing 2–3 teams
- **Daily needs:** View productivity dashboards, manage on-call rotations, identify chronic blockers
- **Pain points:** No visibility into team health without interrupting engineers

## System Architecture

### High-Level Diagram

┌─────────────────────────────────────────────────────────────────────┐
│ USER (Browser) │
└──────────────────────────────┬──────────────────────────────────────┘
│
│ HTTPS
▼
┌─────────────────────────────────────────────────────────────────────┐
│ FRONTEND (Vercel) │
│ React + TypeScript + Tailwind CSS + Vite │
│ - Authentication UI │
│ - Standup feed │
│ - Kanban board │
│ - Dashboard / Analytics │
└──────────────────────────────┬──────────────────────────────────────┘
│
│ REST + WebSocket
▼
┌─────────────────────────────────────────────────────────────────────┐
│ BACKEND (Railway) │
│ Node.js + Express + Socket.io │
│ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │
│ │Auth Routes │ │Standup │ │Ticket │ │Notification│ │
│ │ │ │Routes │ │Routes │ │Service │ │
│ └────────────┘ └────────────┘ └────────────┘ └────────────┘ │
└────────┬─────────────────────────────────────────────┬──────────────┘
│ │
│ Prisma ORM │ HTTPS
▼ ▼
┌─────────────────────────┐ ┌─────────────────────────┐
│ POSTGRESQL (Neon) │ │ GITHUB API │
│ - users │ │ - OAuth │
│ - standups │ │ - Pull Requests │
│ - tickets │ │ - Webhooks │
│ - comments │ │ │
│ - notifications │ │ │
└─────────────────────────┘ └─────────────────────────┘

### Request Flow Example: Submitting a Standup

1. User fills out standup form in React
2. Frontend sends `POST /api/standups` with JWT in headers
3. Backend `authMiddleware` verifies the JWT
4. Standup controller validates input and writes to PostgreSQL via Prisma
5. Backend emits a `standup.created` WebSocket event to subscribed clients
6. Other team members' UIs update in real time
7. Backend returns `201 Created` to original requester


## Database Schema

### Tables Overview

| Table | Purpose |
|-------|---------|
| `users` | User accounts and roles |
| `standups` | Daily standup entries |
| `tickets` | Kanban board items |
| `comments` | Comments on tickets |
| `notifications` | In-app notifications |
| `oncall_shifts` | On-call rotation schedule |

---

### `users`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | UUID | PK | Primary key |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Login identifier |
| `password_hash` | VARCHAR(255) | NOT NULL | bcrypt hash |
| `name` | VARCHAR(100) | NOT NULL | Display name |
| `role` | ENUM | NOT NULL | `engineer`, `tech_lead`, `manager` |
| `github_username` | VARCHAR(100) | NULLABLE | For PR integration |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |

---

### `standups`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, NOT NULL | |
| `yesterday` | TEXT | NOT NULL | What they did |
| `today` | TEXT | NOT NULL | What they plan to do |
| `blockers` | TEXT | NULLABLE | Any blockers |
| `blocker_resolved` | BOOLEAN | DEFAULT FALSE | |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |

**Indexes:** `(user_id, created_at DESC)` for fast feed queries

---

### `tickets`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | UUID | PK | |
| `title` | VARCHAR(200) | NOT NULL | |
| `description` | TEXT | NULLABLE | |
| `status` | ENUM | NOT NULL | `backlog`, `in_progress`, `in_review`, `done` |
| `priority` | ENUM | NOT NULL | `low`, `medium`, `high` |
| `story_points` | INTEGER | NULLABLE | |
| `assignee_id` | UUID | FK → users.id, NULLABLE | |
| `created_by` | UUID | FK → users.id, NOT NULL | |
| `github_pr_url` | VARCHAR(500) | NULLABLE | Linked PR |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |

**Indexes:** `(status)`, `(assignee_id)`

---

### `comments`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | UUID | PK | |
| `ticket_id` | UUID | FK → tickets.id, NOT NULL | |
| `user_id` | UUID | FK → users.id, NOT NULL | |
| `content` | TEXT | NOT NULL | |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |

---

### `notifications`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, NOT NULL | Recipient |
| `type` | ENUM | NOT NULL | `mention`, `pr_review`, `blocker`, `assignment` |
| `content` | TEXT | NOT NULL | |
| `link` | VARCHAR(500) | NULLABLE | URL to relevant resource |
| `read` | BOOLEAN | DEFAULT FALSE | |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |

**Indexes:** `(user_id, read, created_at DESC)`

---

### `oncall_shifts`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, NOT NULL | |
| `start_date` | DATE | NOT NULL | |
| `end_date` | DATE | NOT NULL | |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | |

---

### Entity Relationships
users (1) ──────< (many) standups
users (1) ──────< (many) tickets [as assignee]
users (1) ──────< (many) tickets [as creator]
tickets (1) ────< (many) comments
users (1) ──────< (many) comments
users (1) ──────< (many) notifications
users (1) ──────< (many) oncall_shifts

## API Endpoints

### Convention

- All endpoints prefixed with `/api`
- All requests/responses use JSON
- Authenticated routes require `Authorization: Bearer <JWT>` header
- Standard response codes: 200 (OK), 201 (Created), 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found), 500 (Server Error)

---

### 🔐 Authentication

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/signup` | No | Create new user account |
| POST | `/api/auth/login` | No | Authenticate and receive JWT |
| POST | `/api/auth/logout` | Yes | Invalidate session |
| GET | `/api/auth/me` | Yes | Get current user info |

---

### 🗣️ Standups

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/standups` | Yes | Submit a standup |
| GET | `/api/standups` | Yes | List standups (with filters) |
| GET | `/api/standups/:id` | Yes | Get specific standup |
| PATCH | `/api/standups/:id` | Yes (owner) | Update standup |
| DELETE | `/api/standups/:id` | Yes (owner) | Delete standup |

---

### 📌 Tickets

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/tickets` | Yes | Create ticket |
| GET | `/api/tickets` | Yes | List all tickets (with filters) |
| GET | `/api/tickets/:id` | Yes | Get specific ticket |
| PATCH | `/api/tickets/:id` | Yes | Update ticket (status, assignee, etc.) |
| DELETE | `/api/tickets/:id` | Yes (creator/lead) | Delete ticket |
| POST | `/api/tickets/:id/comments` | Yes | Add comment |

---

### 🔗 GitHub Integration

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/github/oauth/start` | Yes | Begin GitHub OAuth |
| GET | `/api/github/oauth/callback` | No | OAuth callback handler |
| GET | `/api/github/prs` | Yes | List user's open PRs |
| POST | `/api/github/webhook` | No (signed) | Receive GitHub webhook events |

---

### 🔔 Notifications

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/notifications` | Yes | List user's notifications |
| PATCH | `/api/notifications/:id/read` | Yes | Mark as read |
| PATCH | `/api/notifications/read-all` | Yes | Mark all as read |

---

### 📊 Analytics

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/analytics/team` | Yes (lead/manager) | Team metrics |
| GET | `/api/analytics/standups/participation` | Yes (lead/manager) | Standup rate |
| GET | `/api/analytics/tickets/velocity` | Yes (lead/manager) | Ticket throughput |

---

### 📟 On-Call

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/oncall/current` | Yes | Who's on-call right now |
| GET | `/api/oncall/schedule` | Yes | Full rotation schedule |
| POST | `/api/oncall/shifts` | Yes (manager) | Create shift |
| DELETE | `/api/oncall/shifts/:id` | Yes (manager) | Remove shift |

---

### 🔌 WebSocket Events

| Event | Direction | Payload | Description |
|-------|-----------|---------|-------------|
| `notification.new` | Server → Client | `{ id, type, content, link }` | New notification |
| `standup.created` | Server → Client | `{ id, user_id, ... }` | Team member posted standup |
| `ticket.updated` | Server → Client | `{ id, status, ... }` | Ticket changed |
| `subscribe.team` | Client → Server | `{ team_id }` | Subscribe to team events |


## Tech Stack Justification

| Layer | Choice | Why Chosen | Alternatives Considered |
|-------|--------|------------|-------------------------|
| Frontend | React + TypeScript | Most-used framework in industry; TS catches bugs early | Vue, Svelte (smaller ecosystems) |
| Styling | Tailwind CSS | Rapid prototyping, no CSS file management | Styled Components, CSS Modules |
| Build Tool | Vite | 10x faster than Create React App; modern standard | Webpack, Next.js (overkill) |
| Backend | Node.js + Express | Same language as frontend; massive ecosystem | FastAPI (Python), Go (smaller community) |
| ORM | Prisma | Type-safe queries; great DX; auto-generated types | Sequelize, raw SQL |
| Database | PostgreSQL | Industry standard; handles relational data + JSON | MySQL, MongoDB (NoSQL not needed here) |
| Real-time | Socket.io | Mature, easy to use, handles fallbacks | Native WebSockets (more work) |
| Auth | JWT + bcrypt | Stateless; widely understood | Sessions (requires Redis), Auth0 (vendor lock-in) |
| Testing | Jest + Supertest + Playwright | Covers unit, integration, E2E | Vitest, Cypress |
| CI/CD | GitHub Actions | Free for public repos; native GitHub integration | CircleCI, Jenkins |
| Frontend Host | Vercel | Free tier, optimized for React, instant deploys | Netlify, AWS Amplify |
| Backend Host | Railway | Easy Postgres + Node deployment, generous free tier | Heroku (no free tier), AWS (steep learning curve) |
| DB Host | Neon | Serverless Postgres, generous free tier | Supabase, AWS RDS |


## Key Design Decisions

### Decision 1: JWT vs. Session-based Auth
- **Chose:** JWT
- **Why:** Stateless authentication scales easily; no Redis required for v1
- **Trade-off:** Can't revoke tokens before expiration without a blocklist
- **Mitigation:** Short token lifetimes (1 hour), refresh tokens for longer sessions

### Decision 2: Single-tenant vs. Multi-tenant
- **Chose:** Single-tenant for v1
- **Why:** Drastically simpler schema; we're not building SaaS yet
- **Trade-off:** Can't onboard multiple teams without rearchitecture
- **Mitigation:** Document this clearly; revisit in v2

### Decision 3: Polling vs. WebSockets for Real-time
- **Chose:** WebSockets (Socket.io)
- **Why:** Sub-second latency for notifications; better UX
- **Trade-off:** More complex than polling; harder to load-balance
- **Mitigation:** Socket.io handles reconnection automatically

### Decision 4: REST vs. GraphQL
- **Chose:** REST
- **Why:** Simpler to implement; smaller learning curve; well-suited to our resource model
- **Trade-off:** Over-fetching/under-fetching possible
- **Mitigation:** Add query params for field selection if needed

### Decision 5: UUIDs vs. Auto-increment IDs
- **Chose:** UUIDs
- **Why:** No sequential ID enumeration attacks; works across distributed systems
- **Trade-off:** Slightly larger storage size; less human-readable
- **Mitigation:** None needed for our scale


## Risks & Open Questions

### 🚨 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| GitHub API rate limits exceeded | Medium | High | Cache responses, use authenticated requests |
| WebSocket connections drop on free hosting | Medium | Medium | Auto-reconnect logic in client |
| Database schema changes mid-project | High | High | Use Prisma migrations from Day 1 |
| Free tier limits hit during demo | Low | High | Have fallback to local Docker setup |
| JWT secret leaked | Low | Critical | Store in env vars, never commit |

### ❓ Open Questions

1. Should we support OAuth login (Google, GitHub) in v1, or only email/password?
   - **Decision:** Email/password only for v1; OAuth as Week 8 stretch
2. How do we handle timezone differences for standups?
   - **Decision:** Store UTC, display in user's local timezone
3. Should ticket comments support Markdown?
   - **Decision:** Yes — use `react-markdown` for rendering
4. How long should JWT tokens last?
   - **Decision:** 1 hour access tokens, 7 day refresh tokens


## Milestones

### Week 1 (Current): Foundations
- ✅ Repository setup with branch protection
- ✅ Issue templates
- 🔄 Design document (this!)
- ⏳ Project skeletons (frontend + backend)
- ⏳ Docker Compose for local Postgres

### Week 2: Authentication
- User signup + login
- JWT middleware
- Role-based access control
- Auth unit tests

### Week 3: Standups
- Standup CRUD API
- Submission form UI
- Team feed UI

### Week 4: Kanban Board
- Tickets CRUD API
- Drag-and-drop UI
- Comments

### Week 5: GitHub Integration
- OAuth flow
- PR fetching + display
- Webhook handler

### Week 6: Real-time + Notifications
- Socket.io setup
- Notification system
- In-app notification UI

### Week 7: Analytics + On-Call
- Metrics endpoints
- Dashboard charts
- On-call calendar

### Week 8: Polish + Deploy
- E2E tests
- GitHub Actions CI/CD
- Production deployment
- Sentry monitoring
- Case study writeup