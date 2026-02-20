# AGENTS.md — ELIPFORGE (90-Day MVP: B2B)

This repository is the source of truth for implementing ELIPFORGE's 90-day MVP.
Agents must follow the architecture invariants and delivery rules below.

## 0) 90-Day MVP Objective
Deliver a working ELIPFORGE platform with:
- **Single Management Panel** (ONE UI) supporting:
  - **Platform Mode**: manage apps/channels/sites, secrets, rate limits, logs, health
  - **App Mode**: manage B2B business data (products, RFQs, quotes, orders)
  - **App Switcher**: user can switch between multiple apps they own/manage
- **Single Source of Truth DB** (ELIPFORGE DB): all business data lives here (no app-local admin panels)
- **B2B MVP module**:
  - Sprint 4: Product Catalog + RFQ (multi-item)
  - Sprint 5: Quote management + buyer accept/reject + quote→order + idempotency
  - Sprint 6: Orders + hardening + go-live

## 1) Non-Negotiable Architecture Invariants

### 1.1 Isolation model
- **Hard isolation is by `app_id`** in all business tables.
- Cross-app data leakage is a **release blocker**.
- A user can belong to multiple apps; access is controlled by memberships and roles scoped to app.

### 1.2 Clones: same business data, different presentation
- ELIPFORGE supports "clone apps" (presentation variants).
- In MVP: clones are represented as **Sites** under the same app:
  - Example: `toptanbilya.com` and `vitrinxx.com` share the same `app_id`
  - Business data shared (products/RFQ/quote/order shared)
- Presentation differences: theme/config at site level.

### 1.3 Channels: web vs mobile vs others (security + analytics)
- Each app has one or more **Channels**:
  - `web`, `mobile` (later: `admin`, `partner`, `service`)
- **HMAC authentication is performed at channel level** (not app level):
  - `channels.public_key` is used as X-APP-ID
  - `channels.secret_hash` stores the secret securely
- Analytics MUST include `app_id + channel_id` always; `site_id` is optional.

### 1.4 Single Management Panel, two modes
- There is ONE management UI:
  - `/platform/*` routes: platform roles only
  - `/apps/{app_id}/*` routes: app memberships only
- No separate per-app admin panel.

## 2) Identity & Authorization

### 2.1 Roles and scopes
- **Platform roles** (global): `platform_owner`, `platform_admin`, `platform_support`
- **App roles** (scoped to app_id): `app_owner`, `app_admin`, `app_editor`, `app_viewer`

### 2.2 Membership model (MVP)
Tables:
- `users` (global identities)
- `apps`
- `app_memberships` (`user_id`, `app_id`, `role`)

## 3) Security: Channel-Level HMAC (Required)
Headers:
- `X-APP-ID`: channel public_key
- `X-TS`: unix timestamp seconds
- `X-NONCE`: random string
- `X-SIGNATURE`: HMAC-SHA256(secret, canonical_string)

Canonical string (MVP):
{METHOD}\n{PATH_WITH_QUERY}\n{X-TS}\n{X-NONCE}\n{BODY_SHA256_HEX}

Rules:
- Timestamp drift: ±300 seconds
- Nonce replay protection: store nonce in Redis with TTL (>= 10 min), keyed by `channel_id:nonce`
- Body hash: SHA256 of raw body bytes; empty body => SHA256 of empty string
- Constant-time signature compare
- Reject if channel status != active
- If Origin present, validate against channel.allowed_origins (web channels)

Error codes (examples):
- `APP_AUTH_INVALID`, `APP_AUTH_EXPIRED`, `APP_AUTH_REPLAY`, `APP_AUTH_FORBIDDEN_ORIGIN`

## 4) API Standards (Required)
- Versioning: `/api/v1/*`
- Response envelope: `data`, `meta`, `errors`, `trace_id`
- Error structure: `error_code`, `message`, `fields` (optional validation)
- trace_id generated per request and included in response + logs

## 5) Testing Requirements (Release blockers)
Feature tests must cover:
- app_id isolation (A vs B) for all endpoints
- HMAC: valid/invalid/expired/replay
- RBAC and membership checks
- State machine transitions (RFQ/Quote/Order)
- Idempotency on quote accept (no double order creation)

## 6) Suggested Commands (update if repo differs)
Backend (Laravel):
- composer install
- php artisan key:generate
- php artisan migrate --seed
- php artisan test

Frontend (React TS):
- npm install
- npm run lint
- npm test
- npm run build

## 7) Definition of Done
- Tests green in CI
- API contract updated under `docs/codex/03_api_contract.md` if endpoints changed
- No cross-app leakage
- Logs include trace_id and relevant IDs (app_id, channel_id, site_id)

## 8) Backlog
See `docs/codex/02_backlog.md`.
