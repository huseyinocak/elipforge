# Codex Cloud First Task Prompt (Single PR)

Implement Sprint 1–2 foundations.

Deliver:
- Migrations/models/seeders: apps, channels (HMAC), sites, rate_limit_profiles, app_memberships (+ roles if needed)
- Channel-level HMAC middleware (drift ±300s, nonce replay Redis, constant-time compare, origin allowlist)
- Response envelope + stable error_code + trace_id
- Tests: HMAC valid/invalid/expired/replay, origin allowlist, app_id isolation scaffolding
- Skeleton RBAC separation for /platform vs /apps/{app_id} and app switcher endpoint

Update docs/codex/01_architecture.md and 03_api_contract.md as needed.
