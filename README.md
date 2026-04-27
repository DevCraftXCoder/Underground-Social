# Underground Social

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![Cloudflare Workers](https://img.shields.io/badge/Cloudflare_Workers-F38020?style=flat&logo=cloudflare&logoColor=white)
![Hono](https://img.shields.io/badge/Hono-E36002?style=flat&logo=hono&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js_15-000000?style=flat&logo=nextdotjs&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-635BFF?style=flat&logo=stripe&logoColor=white)
![Sentry](https://img.shields.io/badge/Sentry-362D59?style=flat&logo=sentry&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

**Edge-native social music platform for independent and underground artists.**

> Built and operated as a solo full-stack project. Every system — from auth to audio streaming to AI-powered moderation — runs on Cloudflare's edge with zero cold starts and globally distributed latency.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [7 Core Systems](#7-core-systems)
- [API Surface](#api-surface-150-routes-41-route-files)
- [Security Architecture](#security-architecture)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Database Schema Highlights](#database-schema-highlights)
- [Cron Jobs](#cron-jobs)
- [Running This](#running-this)

---

## Architecture

```
Browser
  │
  ▼
Next.js 15  (SSR · App Router · API proxy)
  │
  ▼  Authorization: Bearer <API_KEY>
TypeScript + Hono  (Cloudflare Workers — 41 route files)
  │
  ├── D1 (SQLite relational store — 30+ tables, 58 migrations)
  ├── R2 (audio · covers · avatars · banners · HLS segments · nightly backups)
  ├── Workers AI (track + user recommendations · content moderation · report auto-research)
  ├── Durable Objects (real-time DM WebSockets — per-conversation SQLite isolates)
  ├── Workers Rate Limiting API (4 durable namespaces · cross-isolate · survives restarts)
  ├── Stripe (Underground+ subscription tiers · checkout · webhooks)
  ├── Sentry (error tracking · 10% trace sampling)
  └── Discord Webhook (uptime alerts · first-failure-only)
```

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| API Runtime | TypeScript + Hono | 41 route files, Cloudflare edge runtime |
| Hosting | Cloudflare Workers | Serverless, globally distributed, zero cold starts |
| Database | Cloudflare D1 (SQLite) | 30+ tables, 60 schema migrations |
| File Storage | Cloudflare R2 | Audio, covers, avatars, banners, HLS segments, nightly backups |
| AI | Cloudflare Workers AI | `@cf/baai/bge-base-en-v1.5` recommendations; content moderation |
| Real-time | Durable Objects (DMRoom) | Per-conversation WebSocket + SQLite isolate |
| Rate Limiting | Workers Rate Limiting API | 4 durable namespaces, cross-isolate, restart-safe |
| Payments | Stripe | Subscription tiers + webhooks |
| Error Tracking | Sentry | Exception capture, 10% trace sampling |
| Frontend | Next.js 15 | App Router, RSC, API proxy routes to Worker |
| Validation | Zod | All request/response schemas |
| Auth | Custom JWT + Web Crypto | HS256 — no third-party auth library dependency |

---

## 7 Core Systems

All built in 41 Hono route files:

| System | What's Built |
|--------|-------------|
| **Content** | Tracks (CRUD · draft · publish · HLS streaming) · upload (R2 presigned) · feed · discover · library · search · type-beats · playlists · music links · AI response cache |
| **Social** | Posts (scheduled publishing) · profiles (private/public · verified · handle lookup · insights) · follows · likes · bookmarks · reposts · comments · activity feed · AI recommendations |
| **Collaboration** | Listings CRUD · request workflow (send / respond / withdraw) |
| **Messaging** | DMs (REST + real-time WebSocket via Durable Objects) · message requests · one-time WS auth tickets (60s TTL, atomically consumed) |
| **Notifications** | Paginated list · unread count · mark-read (single + batch) · clear all · admin announcements with per-user dismissal · Web Push VAPID subscribe/unsubscribe |
| **Auth & Identity** | Email/password · Google OAuth · Discord OAuth · JWT rotation · soft delete (30-day grace + GDPR export) · connected accounts |
| **Admin & Platform** | Dashboard stats · user/track management · AI-researched moderation queue · bans · badges · uptime monitoring · error logs · Stripe webhooks · per-track analytics · revenue dashboard |

---

## API Surface (150+ routes, 41 route files)

| Domain | Routes | Auth |
|---|---|---|
| Auth (email, Google, Discord) | 12 | Public |
| Account (delete, recover, export) | 3 | JWT |
| Feed | 3 | Public / JWT |
| Tracks | 12 | JWT |
| Upload | 2 | JWT |
| Library | 3 | JWT |
| Search | 1 | JWT |
| Type Beats | 8 | JWT |
| Playlists | 10 | JWT |
| Music Links | 3 | JWT |
| Recommendations | 2 | JWT |
| Posts | 5 | JWT / Public |
| Profiles | 10 | JWT / Public |
| Social (follow, likes) | 7 | JWT |
| Bookmarks, Reposts, Comments | 7 | JWT |
| Activity | 1 | JWT |
| Collaboration | 10 | JWT |
| Messages + Message Requests | 11 | JWT |
| Real-time DMs (WebSocket) | 2 | One-time ticket |
| Notifications + Announcements | 7 | JWT |
| Push Notifications | 3 | JWT / Public |
| Settings, Blocks, Mutes | 12 | JWT |
| Connected Accounts | 3 | JWT |
| Admin | 18 | API Key |
| Payments (Stripe) | 2 | JWT / Webhook signature |
| Analytics | 5 | JWT / Public |
| Discover | 2 | Public |
| Infrastructure | 1 | Public |

---

## Security Architecture

Authentication and cryptography are implemented using the **Web Crypto API** directly — no Node.js `crypto` module (which breaks in Cloudflare's edge runtime), no third-party JWT libraries.

### Authentication

- **JWT HS256:** 15-minute access tokens + 30-day refresh tokens. Refresh rotation on every use — a stolen refresh token is automatically invalidated on the next legitimate use.
- **Timing-safe login:** Dummy hash comparison always executes for unknown email addresses, preventing email enumeration attacks via response-time differences.
- **OAuth validation:** Google OIDC tokens are checked for `email_verified === true` — unverified Google accounts rejected at login.
- **WebSocket auth:** One-time D1 tickets (60-second TTL, atomically consumed) — credentials never appear in WebSocket query parameters.

### Authorization

- **CSRF protection:** `X-Requested-With: XMLHttpRequest` required on all state-changing routes. Explicit exemptions for auth endpoints, admin API, and Stripe webhooks only.
- **Proxy auth:** Next.js proxy authenticates via a server-side header — verified with a timing-safe comparison. Never reaches client-side code.
- **Tier gating:** Subscription tier is set by server-side JWT middleware from a trusted internal header — never trusted from client input. D1 is authoritative; synced on every Stripe webhook.
- **IDOR prevention:** After validating an R2 key's format, the embedded entity ID is checked against D1 to confirm the requesting user owns the object — prevents cross-user file hijacking on uploads.
- **Block enforcement:** Block checks run in both directions (A→B and B→A) before returning any user data or permitting social actions.
- **Admin isolation:** Admin routes use a separate API key path — completely independent from the user JWT system. A compromised user JWT cannot access any admin endpoint.

### Input & Data Security

- **Zod validation:** Every route validates input schemas before any DB query or business logic runs.
- **SQL safety:** Parameterized D1 queries throughout — no string concatenation in SQL.
- **Content moderation:** Two-layer system — synchronous keyword filter (fast path, pre-insert) + async Workers AI semantic analysis for edits and reports.
- **File validation:** Magic byte checks on uploads — MIME type alone is not trusted.
- **Notification deduplication:** `INSERT OR IGNORE` on all notification inserts — prevents duplicate notifications under race conditions (double-click, concurrent triggers).

### Rate Limiting

| Namespace | Key Scope | Limit | Use Case |
|-----------|-----------|-------|----------|
| Auth | Per IP | 10 req/min | Login, signup, OAuth |
| Email | Per IP | 3 req/hr | Password reset, email verify |
| Messages | Per user | 30 req/min | DM send |
| Creation | Per user (action-scoped) | 10 req/min | Tracks, posts, collabs |

All four limiters are **durable** (survive Worker restarts) and **cross-isolate** (synchronized across CF edge nodes). Action-scoped keys (e.g., `track-create:userId`) prevent one action's rate limit from consuming another's budget.

### Privacy & Compliance

- **Soft delete + hard purge:** Accounts soft-delete with a 30-day grace period. Nightly cron hard-purges expired accounts — cascades through all 30+ related tables and all R2 objects (audio, HLS, covers, avatars, banners).
- **GDPR export:** Full user data export at `/api/account/export`.
- **Error exposure policy:** Internal errors never reach API consumers — generic messages to clients, details to Sentry and internal error log table only.
- **Soft-delete + ban filters:** `deleted_at IS NULL AND is_banned = 0` enforced on every user-facing query.

---

## Key Engineering Decisions

### Edge-first, zero cold starts
All API logic runs on Cloudflare Workers. No traditional server, no Kubernetes, no Dockerized backend. The CDN edge is the compute layer — globally distributed, auto-scaling.

### Custom JWT over third-party auth
Web Crypto API throughout — `import crypto from 'crypto'` breaks in CF Workers. 15-minute access tokens, 30-day refresh tokens with single-use rotation. Timing-safe comparisons on all credential checks.

### Cursor pagination over OFFSET
All list endpoints use composite `<created_at>_<id>` cursors. OFFSET pagination produces inconsistent results under concurrent writes. Cursor-based is stable and predictable.

### Durable Objects for WebSocket DMs
Each DM conversation is its own Durable Object isolate with per-conversation SQLite state. No shared connection pool — each conversation scales independently with zero interference.

### Rate limiting at the edge
Cloudflare Workers Rate Limiting API — cross-isolate, durable, survives Worker restarts. Four namespaced limiters with action-scoped keys to prevent rate budget collision between unrelated actions.

### Soft delete + hard purge
Accounts soft-delete with a 30-day grace period for recovery. A nightly cron job hard-deletes expired accounts — cascading through all 30+ tables and all R2 objects cleanly.

### CSRF at the edge
`X-Requested-With: XMLHttpRequest` required on all state-changing requests. Explicit exemptions for auth, admin, webhooks, and internal callbacks only.

---

## Database Schema Highlights

**30+ tables, 60 migrations**

- `user_profiles` — subscription tier, Stripe IDs, handle, verified status, accent colors, soft-delete
- `tracks` — HLS status, bitrate, waveform, exclusive flag, soft-delete
- `notifications` — `INSERT OR IGNORE` (race-condition-safe), actor_id, typed
- `ws_tickets` — one-time tokens, 60s TTL, atomically consumed
- `user_settings` — 35+ preference columns (theme, audio, privacy, notification toggles)
- `muted_users` — composite PK, optional expiry
- `muted_words` — per-user word filters, max 100
- `push_subscriptions` — VAPID p256dh + auth, unique on endpoint

Indexes on all foreign keys, filter columns, and ORDER BY fields. CHECK constraints on all enum columns. Audit trails (`created_at`, `updated_at`, `deleted_at`) throughout.

---

## Cron Jobs (every 10 minutes)

1. Uptime health checks — parallel fetch of external services
2. Discord alerts — first-failure-only (prevents alert storms on sustained outage)
3. TTL cleanup — error_logs (30d) + uptime_checks (7d)
4. Scheduled post auto-publishing
5. Nightly D1 → R2 backup (midnight UTC, 26 tables)
6. Hard-delete expired soft-deleted accounts (1am UTC, full cascade + R2 cleanup)

---

## Running This

```bash
cd packages/underground-api
npm install

# Local dev
npm run dev          # wrangler dev

# Type check
npm run typecheck    # tsc --noEmit

# Tests
npm run test         # vitest

# Deploy
npm run deploy       # wrangler deploy

# Apply DB migrations
wrangler d1 migrations apply underground-db --remote
```

See `.dev.vars.example` for the full list of required environment variables.

---

## License

MIT — see [LICENSE](LICENSE)
