# ELIPFORGE Context (Quick)

## 90-day MVP
- Single Management Panel
- Single DB (ELIPFORGE DB)
- B2B: Product Catalog + RFQ → Quote → Order
- Isolation: app_id
- Analytics: app_id + channel_id required, site_id optional

## Key concepts
- App: business container (data ownership, memberships, entitlements)
- Channel: web/mobile identity + HMAC keypair (public_key/secret)
- Site: domain/theme clone (optional)
