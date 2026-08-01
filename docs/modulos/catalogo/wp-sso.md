# mod.wp-sso — SSO desde WordPress

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **integration** (desactivable)

## Qué hace

Permite que un usuario **ya autenticado en WordPress** entre a Didacta sin volver a iniciar sesión. El plugin de WordPress (incluido en el módulo: `didacta-sso.php`) firma un JWT HS256 corto (email, nombre, caducidad ≤ 5 min, `jti` único) y redirige al callback de Didacta, que verifica firma, issuer, audiencia y TTL, resuelve al usuario por email (creando un alumno si no existe y el auto-provisionado está activo) y emite la sesión.

## Cómo funciona

- El token es de **un solo uso**: el `jti` se marca como consumido (anti-replay, en Redis con fallback en memoria).
- El secreto HMAC solo lo conocen WordPress y Didacta; nunca viaja al navegador.
- Los emails se normalizan (trim + lowercase) antes de resolver el usuario.
- La redirección final usa la base web endurecida (`WEB_PUBLIC_URL` / allowlist), nunca el header `Host` — evita open-redirect con exfiltración de tokens.
- En WordPress: `DIDACTA_SSO_SECRET` y `DIDACTA_SSO_CALLBACK` en `wp-config.php`, y el botón vía shortcode `[didacta_sso_button]`.

## Dependencias

Ninguna.

## Modelo de datos

Sin tablas propias: el único estado (el `jti` consumido) es efímero y vive en Redis. Los usuarios se crean/resuelven en la tabla de usuarios del core.

## API

- `GET /modules/wp-sso/:tenantSlug/status` — config pública (para el auto-redirect del signin).
- `GET /modules/wp-sso/:tenantSlug/callback?token=` — intercambio del token por sesión (302).
- Configuración admin en `/admin/sso/wp/config` (Community, sin capability).

## Eventos

**Emite**: `wp-sso.signin.success`, `wp-sso.user.provisioned`, `wp-sso.account.linked`. No consume.

## Configuración

Por tenant desde el panel (Administración → SSO → WordPress): secreto compartido (cifrado), issuer/audience opcionales, auto-provisionado y auto-redirect.
