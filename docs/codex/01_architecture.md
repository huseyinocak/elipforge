# Architecture — App → Channel → Site (MVP)

## 1) High-level
- ELIPFORGE hosts the **single source of truth** database and the only Management Panel.
- Independent clients (web/mobile) are presentation-only; they call ELIPFORGE API with channel-level HMAC.

## 2) Isolation (must-have)
- All business tables include `app_id`.
- All queries MUST scope by `app_id`.
- Cross-app leakage is a release blocker.

## 3) App, Channel, Site
- **App**: business container (data ownership, memberships, entitlements).
- **Channel**: web/mobile client identity; owns HMAC keys; required for analytics segmentation.
- **Site**: optional presentation instance (domain/theme). Multiple sites can exist under the same app.

## 4) Clones
- Clone apps are implemented as multiple **Sites** under the same `app_id`.
- Business data is shared across clones.
- Analytics can be segmented by `channel_id` and (optionally) `site_id`.

## 5) Security (channel-level HMAC)
- Middleware verifies HMAC signature using `channels.public_key` and secret_hash (secret shown once).
- Drift ±300 seconds; nonce replay protection via Redis.
- Origin allowlist applied when Origin header present (web).

## 6) Single Management Panel (two modes)
- Platform Mode: manage apps/channels/sites, secrets, rate profiles, logs/health.
- App Mode: manage B2B data (products/RFQs/quotes/orders) for selected app.
- App Switcher uses `app_memberships` to allow a user to switch apps.

## 7) Analytics
- Event ingestion endpoint stores:
  - app_id (required), channel_id (required), site_id (nullable)
- Reporting provides:
  - totals by app
  - breakdown by channel (web vs mobile)
  - breakdown by site/domain (clone segmentation)
