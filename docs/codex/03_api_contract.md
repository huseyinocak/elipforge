# API Contract — v1 (MVP) — ELIPFORGE B2B

This contract is designed to be **agent-implementable**. It defines:
- Security: channel-level HMAC (required)
- Envelope: stable response schema + error codes
- RBAC + scopes
- Entities + state machines (RFQ/Quote/Order)
- Endpoints + request/response examples

> Source of truth rules: see `/AGENTS.md` (app_id isolation, single panel, clones via sites).

---

## 0) Terminology and scopes

### Scope keys (always)
- `app_id`: hard isolation boundary. Never cross-leak.
- `channel_id`: web/mobile boundary and HMAC identity.
- `site_id`: optional (domain/theme clone). Used primarily for analytics segmentation.

### Panel modes (one UI, two areas)
- Platform Mode routes (conceptual): `/platform/*` (platform roles)
- App Mode routes (conceptual): `/apps/{app_id}/*` (app memberships)

API paths below use `/api/v1/...`. Authorization decides whether the caller is platform-scoped or app-scoped.

---

## 1) Security — Channel-level HMAC (Required)

### 1.1 Headers
- `X-APP-ID`: channel public_key (maps to `channels.public_key`)
- `X-TS`: unix timestamp (seconds)
- `X-NONCE`: random string (recommended 16–32 chars)
- `X-SIGNATURE`: hex HMAC-SHA256(signature_key, canonical_string)

### 1.2 Canonical string
```
{METHOD}\n{PATH_WITH_QUERY}\n{X-TS}\n{X-NONCE}\n{BODY_SHA256_HEX}
```
- `METHOD`: uppercase (GET/POST/PATCH/DELETE)
- `PATH_WITH_QUERY`: as received (e.g. `/api/v1/rfqs?page=1`)
- `BODY_SHA256_HEX`: SHA-256 of raw body bytes; empty body => SHA-256 of empty string

### 1.3 Validation rules
- Timestamp drift: ±300 seconds (configurable)
- Nonce replay protection: store `(channel_id, nonce)` in Redis TTL >= 10 minutes
- Constant-time compare for signatures
- Channel status must be `active`
- Origin allowlist:
  - If request includes `Origin`, it must match one of `channels.allowed_origins` (web channels)

### 1.4 Common HMAC errors
- `APP_AUTH_INVALID` (bad signature / unknown channel)
- `APP_AUTH_EXPIRED` (timestamp out of window)
- `APP_AUTH_REPLAY` (nonce reused)
- `APP_AUTH_FORBIDDEN_ORIGIN` (Origin not allowlisted)
- `APP_AUTH_CHANNEL_INACTIVE` (suspended/revoked)

---

## 2) Response envelope and error model (Required)

### 2.1 Success envelope
```json
{
  "data": {},
  "meta": { "trace_id": "trc_..." },
  "errors": []
}
```

### 2.2 Error envelope
```json
{
  "data": null,
  "meta": { "trace_id": "trc_..." },
  "errors": [
    {
      "error_code": "VALIDATION_ERROR",
      "message": "Validation failed",
      "fields": { "name": ["The name field is required."] }
    }
  ]
}
```

### 2.3 Error code conventions (MVP)
- Auth:
  - `APP_AUTH_*` (HMAC)
  - `USER_AUTH_REQUIRED`, `USER_AUTH_INVALID`
- Authorization:
  - `FORBIDDEN_PLATFORM_ROLE`, `FORBIDDEN_APP_ROLE`, `FORBIDDEN_MEMBERSHIP`
- Not found:
  - `NOT_FOUND`
- Validation:
  - `VALIDATION_ERROR`
- State machine:
  - `INVALID_STATE_TRANSITION`
- Idempotency:
  - `IDEMPOTENCY_REPLAY`

---

## 3) Authorization (RBAC)

### 3.1 Roles
**Platform roles (global)**
- `platform_owner`
- `platform_admin`
- `platform_support` (read-only ops)

**App roles (scoped to app_id)**
- `app_owner`
- `app_admin`
- `app_editor`
- `app_viewer`

### 3.2 Minimal permission matrix (MVP)
| Area | Action | Roles |
|---|---|---|
| Platform | Manage apps/channels/sites | platform_owner, platform_admin |
| Platform | View logs/health | platform_owner, platform_admin, platform_support |
| App | Manage products | app_owner, app_admin, app_editor |
| App | View products | app_owner, app_admin, app_editor, app_viewer |
| App | Manage quotes/orders (admin side) | app_owner, app_admin |
| App | View RFQs/quotes/orders | app_owner, app_admin, app_editor, app_viewer |

> Buyer actions (RFQ create, quote accept, view own RFQs) are controlled by **user ownership**, not app roles.

---

## 4) Entities and state machines (MVP)

### 4.1 Product
- `id`, `app_id`
- `name`, `sku?`, `description?`
- `status`: `draft|active`
- `price?` optional (RFQ-centric apps may not show price)

### 4.2 RFQ
- `id`, `app_id`, `channel_id`, `site_id?`
- `buyer_user_id` (or guest token if enabled later)
- `status`: `submitted|quoted|cancelled|expired|closed`
- `notes?`

### 4.3 RFQItem
- `id`, `app_id`, `rfq_id`
- `product_id?` (nullable; allow free-text line)
- `name_snapshot`, `quantity`, `unit`

### 4.4 Quote
- `id`, `app_id`, `rfq_id`
- `status`: `draft|sent|updated|withdrawn|accepted|rejected|expired`
- `valid_until?`

### 4.5 QuoteItem
- `id`, `app_id`, `quote_id`, `rfq_item_id`
- `unit_price`, `currency`, `lead_time?`, `notes?`

### 4.6 Order
- `id`, `app_id`, `channel_id`, `site_id?`
- `source`: `rfq_quote|online_sales`
- `status`: `created|confirmed|cancelled` (+ module-gated: `pending_payment|paid|processing|shipped|delivered`)
- `buyer_user_id`

### 4.7 OrderItem
- `id`, `app_id`, `order_id`
- `product_id?`, `name_snapshot`, `quantity`, `unit_price`, `currency`

### 4.8 Transition rules (minimal)
- RFQ: `submitted` -> `quoted` (when first quote sent), `quoted` -> `closed` (on accepted quote)
- Quote: only `sent|updated` can be accepted/rejected
- Quote after accepted/rejected cannot be modified/withdrawn
- Order created once per accepted quote (idempotent accept)

---

## 5) Endpoint groups (v1)

All endpoints require channel HMAC.
Some endpoints require user auth (panel and buyer flows).

### 5.1 Health
#### GET /api/v1/health
**Auth:** HMAC only
**Response:**
```json
{ "data": { "ok": true }, "meta": { "trace_id": "..." }, "errors": [] }
```

---

## 6) Platform Mode (Apps/Channels/Sites)

> These endpoints are called by the Management Panel in Platform Mode.
> **Auth:** HMAC + User auth + platform role.

### 6.1 Apps
#### POST /api/v1/platform/apps
Create an app container.
**Request:**
```json
{ "name": "toptanbilya", "slug": "toptanbilya", "owner_user_id": "uuid" }
```
**Response data:**
```json
{ "id": "uuid", "name": "toptanbilya", "slug": "toptanbilya", "status": "active" }
```

#### GET /api/v1/platform/apps
List apps (platform scope).
Query: `?search=&status=&page=1`
Response data: `{ "items": [...], "page": 1, "per_page": 20, "total": 123 }`

#### PATCH /api/v1/platform/apps/{id}/status
Request:
```json
{ "status": "suspended" }
```

### 6.2 Channels
#### POST /api/v1/platform/channels
Create channel and return **secret once**.
Request:
```json
{
  "app_id": "uuid",
  "type": "web",
  "name": "Toptanbilya Web",
  "allowed_origins": ["https://toptanbilya.com"],
  "rate_limit_profile_id": "uuid"
}
```
Response data:
```json
{
  "id": "uuid",
  "app_id": "uuid",
  "type": "web",
  "name": "Toptanbilya Web",
  "public_key": "pk_...",
  "secret": "ONLY_SHOWN_ONCE",
  "status": "active"
}
```

#### POST /api/v1/platform/channels/{id}/rotate-secret
Response data includes new secret once.

#### PATCH /api/v1/platform/channels/{id}
Update origins, rate profile, status.

### 6.3 Sites (clone domains)
#### POST /api/v1/platform/sites
Request:
```json
{
  "app_id": "uuid",
  "channel_id": "uuid",
  "domain": "toptanbilya.com",
  "theme_id": "theme_default",
  "site_settings": { "brand": "Toptanbilya" }
}
```

#### GET /api/v1/platform/sites?app_id=...
List sites for an app.

---

## 7) App Mode (Panel) — Products / RFQ view / Quotes / Orders

> These endpoints are called by the Management Panel in App Mode.
> **Auth:** HMAC + User auth + app membership role (scoped to app_id).

### 7.1 Products (CRUD)
#### POST /api/v1/products
Request:
```json
{ "name": "Bilya", "status": "active", "sku": "BLY-001", "description": "..." }
```
Response: created product.

#### GET /api/v1/products?search=&status=&page=1
List products (app scoped).

#### PATCH /api/v1/products/{id}
Update.

#### DELETE /api/v1/products/{id}
Soft delete recommended.

### 7.2 RFQ (Panel view)
#### GET /api/v1/rfqs?status=&from=&to=&page=1
List RFQs (app scoped, role-gated).

#### GET /api/v1/rfqs/{id}
RFQ detail including items.

### 7.3 Quotes (Panel)
#### POST /api/v1/rfqs/{id}/quotes
Request:
```json
{
  "valid_until": "2026-03-15",
  "items": [
    { "rfq_item_id": "uuid", "unit_price": 120.50, "currency": "TRY", "notes": "" }
  ]
}
```

#### POST /api/v1/quotes/{id}/send
Sets status to `sent`, triggers notification placeholder.

#### PATCH /api/v1/quotes/{id}
Revise quote (status -> `updated`).

#### POST /api/v1/quotes/{id}/withdraw
Withdraw quote.

### 7.4 Orders (Panel)
#### GET /api/v1/orders?status=&page=1
#### GET /api/v1/orders/{id}
#### POST /api/v1/orders/{id}/confirm
#### POST /api/v1/orders/{id}/cancel

---

## 8) Buyer endpoints — Catalog / RFQ / Quotes / Orders

> Buyer endpoints are used by independent web/mobile clients.
> **Auth:** HMAC + User auth (buyer session).
> App scope comes from channel->app mapping; user scope from auth.

### 8.1 Catalog
#### GET /api/v1/catalog/products?search=&page=1
Only `active` products.

#### GET /api/v1/catalog/products/{id}

### 8.2 RFQ (create/list/detail)
#### POST /api/v1/rfqs
Request (multi-item):
```json
{
  "notes": "Need prices",
  "items": [
    { "product_id": "uuid", "quantity": 10, "unit": "pcs" },
    { "name": "Custom item", "quantity": 5, "unit": "kg" }
  ]
}
```
Rules:
- sets rfq.status = `submitted`
- stores app_id, channel_id, and site_id if resolvable
- emits analytics event `rfq_created`

#### GET /api/v1/rfqs
Buyer list (own RFQs only).

#### GET /api/v1/rfqs/{id}
Buyer detail (ownership enforced).

### 8.3 Quotes (buyer)
#### GET /api/v1/rfqs/{id}/quotes
List quotes for buyer's RFQ.

#### POST /api/v1/quotes/{id}/accept
Headers:
- `Idempotency-Key: <uuid>`
Rules:
- only `sent|updated` can be accepted
- create order exactly once
- set quote.status = `accepted`, rfq.status = `closed`
Error:
- `IDEMPOTENCY_REPLAY` if replay with different payload (or `OK` with same)

#### POST /api/v1/quotes/{id}/reject
Sets quote.status = `rejected`.

### 8.4 Orders (buyer)
#### GET /api/v1/orders
Buyer list (own orders only).

#### GET /api/v1/orders/{id}

---

## 9) Analytics ingestion (event-based)

### POST /api/v1/analytics/events
**Auth:** HMAC (user optional)
Request:
```json
{
  "event_name": "page_view",
  "path": "/products/123",
  "screen": null,
  "session_id": "s_abc",
  "meta": { "ref": "google" }
}
```
Server behavior:
- attaches `app_id` and `channel_id` from auth context
- derives `site_id` from Host/domain mapping when possible
- stores user_id if authenticated

---

## 10) Indexes and performance (MVP)
- `products(app_id, status, name)` index for listing/search
- `rfqs(app_id, created_at)` and `rfqs(app_id, status, created_at)`
- `analytics_events(app_id, channel_id, created_at)` and optionally `(app_id, site_id, created_at)`

---

## 11) Test checklist (MVP)
- HMAC valid/invalid/expired/replay + origin allowlist
- app_id isolation for products/rfqs/quotes/orders
- buyer ownership for rfqs/quotes/orders
- quote accept idempotency (no double order)
- invalid transitions return `INVALID_STATE_TRANSITION`

