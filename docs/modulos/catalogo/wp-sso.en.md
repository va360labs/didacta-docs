# mod.wp-sso — WordPress SSO

<span class="didacta-chip didacta-chip--community">Community</span> · **Integration** category (can be disabled)

## What it does

It lets a user **already authenticated in WordPress** enter Didacta without signing in again. The WordPress plugin (bundled with the module: `didacta-sso.php`) signs a short HS256 JWT (email, name, expiry ≤ 5 min, unique `jti`) and redirects to Didacta's callback, which verifies the signature, issuer, audience and TTL, resolves the user by email (creating a student if none exists and auto-provisioning is enabled) and issues the session.

## How it works

- The token is **single use**: the `jti` is marked as consumed (anti-replay, in Redis with an in-memory fallback).
- The HMAC secret is known only to WordPress and Didacta; it never reaches the browser.
- Email addresses are normalised (trimmed + lowercased) before the user is resolved.
- The final redirect uses the hardened web base (`WEB_PUBLIC_URL` / allowlist), never the `Host` header — this prevents open redirects with token exfiltration.
- In WordPress: `DIDACTA_SSO_SECRET` and `DIDACTA_SSO_CALLBACK` in `wp-config.php`, and the button through the `[didacta_sso_button]` shortcode.

## Dependencies

None.

## Data model

No tables of its own: the only state (the consumed `jti`) is ephemeral and lives in Redis. Users are created/resolved in the core users table.

## API

- `GET /modules/wp-sso/:tenantSlug/status` — public configuration (for the sign-in auto-redirect).
- `GET /modules/wp-sso/:tenantSlug/callback?token=` — exchanging the token for a session (302).
- Admin configuration at `/admin/sso/wp/config` (Community, no capability required).

## Events

**Emits**: `wp-sso.signin.success`, `wp-sso.user.provisioned`, `wp-sso.account.linked`. It consumes none.

## Configuration

Per tenant from the panel (Administration → Identity (SSO) → WordPress): shared secret (encrypted), optional issuer/audience, auto-provisioning and auto-redirect.
