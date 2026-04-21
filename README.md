# Underground Social

**Edge-native social music platform for independent and underground artists.**

> Built and operated as a solo full-stack project. Every system — from auth to audio streaming to AI-powered moderation — runs on edge infrastructure with zero cold starts.

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

| Layer | Technology |
|---|---|
| API Runtime | TypeScript + Hono, Cloudflare Workers |
| Database | Cloudflare D1 (SQLite, 30+ tables, 58 migrations) |
| File Storage | Cloudflare R2 |
| AI | Cloudflare Workers AI |
| Real-time | Durable Objects (DMRoom WebSocket) |
| Auth | Custom JWT HS256 + Google OAuth + Discord OAuth |
| Payments | Stripe (subscription tiers + webhooks) |
| Frontend | Next.js 15, App Router, SSR |
| Error Tracking | Sentry |
| Monitoring | Cron-based uptime checks + Discord alerts |

---

## Systems

### 1. Content System
- Tracks — CRUD, drafts, publish, audio streaming, HLS multi-bitrate transcoding
- Type Beats — genre / mood / BPM marketplace with likes and streaming
- Playlists — CRUD, reorder, featured, public/private
- Upload — R2 pre-signed URLs + direct upload; client → R2 → webhook → metadata extraction
- Library — liked tracks, listening history, downloads
- Search — full-text with ranking
- AI Cache — cached AI-generated responses

### 2. Social System
- Feed — following-only, all posts, RSS; cursor-paginated
- Posts — CRUD, scheduled publishing, drafts, status state machine
- Profiles — public/private, verified badges, accent colors, social links, handle lookup, insights
- Follow system — follow/unfollow, follow requests for private accounts
- Likes, bookmarks, reposts, comments
- Activity feed (uploads, likes, follows)

### 3. Collaboration System
- Listings — CRUD with open/closed/filled status
- Requests — send, respond (accept/reject), withdraw; full state machine

### 4. Messaging System
- DM conversations — inbox, history, send, delete, mark-read
- Real-time WebSocket via Durable Objects (one DO isolate per conversation)
- One-time WS ticket auth — 60s TTL, atomically consumed from D1; no JWT in query params
- Message requests — stranger approval flow; accept/decline

### 5. Notification System
- List, unread count, mark-read (single + batch), clear all
- Triggered by: likes, comments, follows, reposts, collab requests, DMs
- Web Push VAPID subscribe/unsubscribe
- Admin announcements with per-user dismissal

### 6. Auth & Identity System
- Email/password registration with verification
- Password reset flow (forgot + reset)
- Google OAuth + Discord OAuth
- JWT HS256 — 15-minute access token + 30-day refresh token with rotation
- Rate limiting: 10/min per IP (auth), 3/hr per IP (email)
- Account soft-delete with 30-day grace + GDPR data export
- Connected accounts (link/unlink OAuth providers)

### 7. Admin & Platform System
- Platform-wide stats, follower analytics
- User management: ban, unban, verify, delete with full R2 cascade cleanup
- Track management: list, admin delete with R2 + HLS cleanup
- Content reports with AI auto-research (gathers context for moderators)
- Nightly D1 → R2 backup (26 tables)
- Scheduled post auto-publishing (cron)
- Hard-delete expired soft-deleted accounts (30-day grace, full cascade)
- Error logging to D1 + Sentry; uptime monitoring with Discord alerts

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
| Bookmarks | 3 | JWT |
| Reposts | 1 | JWT |
| Comments | 3 | JWT |
| Activity | 1 | JWT |
| Collaboration | 10 | JWT |
| Messages + Requests | 11 | JWT |
| Real-time DMs (WS) | 2 | Ticket |
| Notifications + Announcements | 7 | JWT |
| Push Notifications | 3 | JWT / Public |
| Settings, Blocks, Mutes | 12 | JWT |
| Connected Accounts | 3 | JWT |
| Admin | 18 | API Key |
| Payments (Stripe) | 2 | JWT / Signature |
| Analytics | 5 | JWT / Public |
| Discover | 2 | Public |
| Infrastructure | 1 | Public |

---

## Key Engineering Decisions

### Edge-first, zero cold starts
All API logic runs on Cloudflare Workers. No traditional server, no Kubernetes, no Dockerized backend. The CDN edge *is* the compute layer — globally distributed, auto-scaling.

### Custom JWT over third-party auth
Web Crypto API throughout (no `import crypto from 'crypto'` — that breaks in CF Workers). 15-minute access tokens, 30-day refresh tokens with rotation. Timing-safe comparisons on all credential checks.

### Cursor pagination over OFFSET
All list endpoints use composite `<created_at>_<id>` cursors. OFFSET pagination produces inconsistent results under concurrent writes. Cursor-based is stable.

### Durable Objects for WebSocket DMs
Each DM conversation is its own Durable Object isolate with per-conversation SQLite state. No shared connection pool — each conversation scales independently.

### Rate limiting at the edge
Cloudflare Workers Rate Limiting API — cross-isolate, durable, survives Worker restarts. Four namespaced limiters with action-scoped keys to prevent budget collision between unrelated actions.

### Soft delete + hard purge
Accounts soft-delete with a 30-day grace period for recovery. A nightly cron job hard-deletes expired accounts, cascading through all 30+ related tables and all R2 objects (audio files, HLS segments, covers, avatars, banners).

### CSRF protection at the edge
`X-Requested-With: XMLHttpRequest` required on all state-changing requests. Explicit exemptions for auth routes, admin routes, Stripe webhooks, and internal HLS callbacks.

---

## Database Schema Highlights

**30+ tables, 58 migrations**

- `user_profiles` — subscription tier, Stripe IDs, handle, verified status, accent colors, soft-delete
- `tracks` — HLS status, bitrate, waveform, exclusive flag, soft-delete
- `notifications` — INSERT OR IGNORE (race condition prevention), actor_id, type
- `ws_tickets` — one-time tokens, 60s TTL, atomically consumed
- `user_settings` — 35+ preference columns (theme, audio, privacy, notification toggles)
- `muted_users` — composite PK, optional expiry
- `muted_words` — per-user word filters, max 100
- `push_subscriptions` — VAPID p256dh + auth, unique on endpoint

Indexes on all foreign keys, filter columns, and ORDER BY fields.
CHECK constraints on all enum columns. Audit trails (`created_at`, `updated_at`, `deleted_at`) throughout.

---

## Security

See [SECURITY.md](SECURITY.md) for the full policy.

- **CSRF** — `X-Requested-With` header required on all state-changing requests
- **JWT rotation** — refresh tokens are single-use; each use issues a new token and revokes the old
- **Timing-safe auth** — dummy hash comparisons on non-existent users (prevents email enumeration)
- **IDOR prevention** — R2 key ownership validated against D1 before writes; embedded ID checked against `user_id`
- **Soft-delete + ban filters** — `deleted_at IS NULL AND is_banned = 0` on all user-facing queries
- **Rate limiting** — durable, cross-isolate, action-scoped keys
- **WebSocket auth** — one-time D1 tickets, no credentials in query params
- **Admin isolation** — admin routes use separate API key auth, never JWT
- **Content moderation** — sync moderation (fast, no AI) on creates; async AI moderation on edits
- **File validation** — magic byte checks on uploads; MIME type alone is not trusted

---

## Cron Jobs (every 10 minutes)

1. Uptime health checks — parallel fetch of external services
2. Discord alerts — first-failure-only to prevent noise
3. TTL cleanup — error_logs (30d) + uptime_checks (7d)
4. Scheduled post auto-publishing
5. Nightly D1 → R2 backup (midnight UTC, 26 tables)
6. Hard-delete expired soft-deleted accounts (1am UTC, full cascade + R2 cleanup)

---

## License

MIT — see [LICENSE](LICENSE)
