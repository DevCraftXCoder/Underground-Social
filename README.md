# Underground Social

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![Cloudflare Workers](https://img.shields.io/badge/Cloudflare_Workers-F38020?style=flat&logo=cloudflare&logoColor=white)
![Hono](https://img.shields.io/badge/Hono-E36002?style=flat&logo=hono&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-635BFF?style=flat&logo=stripe&logoColor=white)

**Edge-native social music platform for independent and underground artists.**

> A full social platform built entirely on Cloudflare's edge network — 42 route files, 84 D1 migrations, 30+ tables, real-time WebSocket DMs via Durable Objects, zero cold starts globally.

## Architecture

```
Browser
  └── Next.js 15 (SSR + API proxy)
        └── Hono Worker (Cloudflare Workers)
              ├── D1 SQLite (84 migrations, 30+ tables)
              ├── R2 (audio, covers, HLS segments, nightly backups)
              ├── Durable Objects (real-time DM WebSockets)
              ├── Workers AI (recommendations, content moderation)
              ├── Workers Rate Limiting (4 namespaces, cross-isolate)
              └── Stripe (Underground+ subscriptions)
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| API | TypeScript + Hono (42 route files) |
| Hosting | Cloudflare Workers — globally distributed, no cold starts |
| Database | Cloudflare D1 (SQLite) — 84 migrations, 30+ tables |
| File Storage | Cloudflare R2 — audio, covers, HLS segments, nightly backups |
| AI | Cloudflare Workers AI — recommendations + content moderation |
| Real-time | Durable Objects — per-conversation WebSocket isolates |
| Rate Limiting | Workers Rate Limiting API — 4 durable namespaces, cross-isolate |
| Payments | Stripe — Underground+ subscription tiers |
| Error Tracking | Sentry — 10% trace sampling |
| Auth | Custom JWT HS256 + Google OAuth + Discord OAuth |

## Systems Built

1. **Content System** — tracks, playlists, type beats, HLS multi-bitrate streaming, R2 uploads
2. **Social System** — feed, posts, follows, likes, reposts, bookmarks, comments, activity
3. **Collaboration System** — listings CRUD + collab request workflow
4. **Messaging System** — DM inbox + real-time WebSocket via Durable Objects + message requests
5. **Notification System** — in-app, Web Push (VAPID), announcements, dismissals
6. **Auth & Identity System** — JWT rotation, Google/Discord OAuth, GDPR export, soft delete
7. **Admin & Platform System** — moderation queue, automated research, uptime monitoring, nightly backups

## Key Engineering Details

- **Rate limiting** via Workers Rate Limiting API — 4 durable namespaces (auth/email/message/creation), cross-isolate, survives restarts
- **Real-time DMs** via Durable Objects — each conversation gets its own SQLite isolate + WebSocket handler
- **HLS multi-bitrate audio** streaming with R2 edge cache and Cache API TTL
- **GDPR-compliant account deletion** — soft delete + 30-day hard purge cron, cascades through 30+ tables + R2
- **Content moderation pipeline** — sync keyword filter + async Workers LLM analysis
- **Cursor-based pagination** everywhere — no OFFSET, stable under concurrent writes
- **CSRF protection** — `X-Requested-With` header required on all state-changing routes
- **Web Push notifications** — VAPID key subscribe/unsubscribe built, delivery pipeline ready

## Scale

| Metric | Count |
|--------|-------|
| Route files | 42 |
| D1 migrations | 84 |
| D1 tables | 30+ |
| Auth providers | 3 (email, Google, Discord) |
| Rate limiter namespaces | 4 (cross-isolate, durable) |
| Cron jobs | 6 (uptime, cleanup, backups, publishing, hard-delete, TTL) |

