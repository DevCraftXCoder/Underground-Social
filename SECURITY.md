# Security Policy

## Reporting a Vulnerability

Do **not** open a public GitHub issue for security vulnerabilities.

Contact via GitHub: [@DevCraftXCoder](https://github.com/DevCraftXCoder)

Response time: 72 hours for acknowledgment, 7 days for critical issues.

---

## Security Architecture

### Authentication

- JWT HS256 — 15-minute access tokens, 30-day rotating refresh tokens
- Refresh tokens are single-use: each rotation issues a new token and revokes the prior
- Web Crypto API throughout — `crypto.subtle` for hashing, `crypto.subtle.timingSafeEqual` for all comparisons
- Dummy hash comparison on login when user doesn't exist — prevents email enumeration via timing
- OAuth `email_verified` field enforced for Google OIDC — unverified emails rejected
- WebSocket auth via one-time D1 tickets (60s TTL, atomically consumed) — no credentials in query params

### Authorization

- All protected routes validate `userId` from JWT middleware — client-supplied user IDs not trusted
- R2 key ownership validated against D1 before writes: embedded entity ID checked against `user_id`
- Admin routes isolated under `X-Admin-Key` header auth — completely separate from JWT layer
- Block checks (`isBlocked()`) are bidirectional on all social actions
- Soft-delete + ban filters applied on all user-facing queries: `deleted_at IS NULL AND is_banned = 0`

### Transport & Input

- CSRF: `X-Requested-With: XMLHttpRequest` required on all state-changing requests
- Explicit CSRF exemptions: auth routes, admin routes, Stripe webhook, HLS transcoder callback
- Zod validation on all request bodies — malformed requests rejected at the boundary
- File uploads: magic byte validation (server-side); MIME type from client not trusted
- Content-length limits enforced before parsing large request bodies

### Data Protection

- Secrets in environment variables only — never in application code or logs
- Secret values logged as `set (N chars)` only — raw values never logged
- Soft-delete with hard purge maintains audit trail while honoring deletion requests
- GDPR: `GET /api/account/export` for data access; `DELETE /api/account` initiates 30-day soft-delete with hard purge

### Rate Limiting

Four durable, cross-isolate rate limiters (Cloudflare Workers Rate Limiting API):

| Limiter | Key Format | Limit |
|---|---|---|
| AUTH | `{ip}:{action}` | 10 / min |
| EMAIL | `{ip}:{action}` | 3 / hr |
| MESSAGE | `dm-send:{userId}` | 30 / min |
| CREATION | `{action}:{userId}` | 10 / min |

Action-scoped keys prevent budget collision between unrelated operations.

### Infrastructure

- All local services bound to `127.0.0.1` — no direct public port exposure
- External access via Cloudflare Named Tunnels only
- DDoS mitigation and TLS termination at Cloudflare edge

---

## Supported Versions

This repository documents a production architecture. The security controls described here apply to the deployed platform.
