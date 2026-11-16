# Autenticación

La API usa **credenciales stateless en la cabecera `Authorization`**: no pone cookies de sesión ni guarda estado de servidor para autenticarte. Según quién llama hay cuatro credenciales distintas:

| Quién | Credencial | Cabecera | Dónde vale |
| --- | --- | --- | --- |
| Una persona (panel, app, scripts) | JWT de sesión | `Authorization: Bearer <accessToken>` | Casi toda la API, según su rol |
| Un sistema externo (tienda, CRM, automatización) | API key `lmsk_…` | `Authorization: ApiKey lmsk_…` | Solo `/inscribe`, `/integrations` y `/community-api`, según sus scopes |
| Un agente de IA | Clave de agente (API key con scopes `agent:*`) | `Authorization: Bearer lmsk_…` o `ApiKey lmsk_…` | Solo el servidor MCP (`/mcp`) — ver [MCP](mcp.md) |
| El plano de control de la instancia (Didacta Cloud o tu propio script) | Credencial de provisioning `didprov_…` | `Authorization: Provisioning didprov_…` | Una lista blanca de rutas sobre organizaciones — ver [más abajo](#3-credencial-de-provisioning) |

## 1. Sesiones de usuario (JWT)

```bash
# Login
curl -X POST https://tu-instancia/api/v1/auth/signin \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "..."}'
# → {
#     "tokens": { "accessToken": "...", "refreshToken": "...", "expiresIn": 3600 },
#     "mfaRequired": false,
#     "user": { "id": "…", "email": "…", "name": "…", "avatarUrl": null,
#               "tenantId": "…", "tenantSlug": "demo" }
#   }

# Llamadas autenticadas
curl -H "Authorization: Bearer <accessToken>" https://tu-instancia/api/v1/me/profile

# Renovar → { "tokens": { … } }
curl -X POST https://tu-instancia/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken": "..."}'
```

Detalles del token:

- Firmado HS256 con `AUTH_SECRET`. El **access token dura 1 hora** y el **refresh token 30 días**. Claims: usuario (`sub`), `tenantId`, `roles[]`, `mfaVerified` y `sid` (id de sesión).
- **Revocación inmediata.** El `sid` referencia una fila de sesión en BD. Cerrar una sesión desde `/me/security/sessions`, suspender la cuenta o suspender el aula corta el acceso sin esperar a que caduque el token: en cada petición se comprueba el estado de la cuenta y la API responde **401** con `code` `session_revoked`, `account_suspended` o `account_deleted` (las rutas `/auth/*` quedan fuera para que una cuenta suspendida pueda cerrar sesión).
- `POST /auth/forgot-password` responde 200 siempre (anti-enumeración de usuarios).
- **No hace falta decir en qué organización entras.** El login resuelve el aula por el dominio de la petición; `tenantSlug` es opcional y solo se necesita si el dominio no está mapeado. Sin dominio ni slug, se busca el email entre las organizaciones activas: si está en más de una, responde `AMBIGUOUS_TENANT` con los slugs candidatos.

### Límite de intentos de acceso

`signin`, `signup`, `forgot-password`, `reset-password`, el reenvío de enlace, la verificación MFA y el canje de acceso de soporte llevan un **limitador propio, más estricto que el general**: por defecto 30 intentos por minuto y por IP, y 10 fallos en 15 minutos por cuenta. Al superarlo responde **429** con `code: "AUTH_RATE_LIMITED"` y `retryAfterSeconds`; el cuerpo es idéntico exista o no la cuenta. Se ajusta con `RATE_LIMIT_LOGIN_IP_MAX`, `RATE_LIMIT_LOGIN_IP_WINDOW_SECONDS`, `RATE_LIMIT_LOGIN_ACCOUNT_MAX` y `RATE_LIMIT_LOGIN_ACCOUNT_WINDOW_SECONDS`. Si Redis no está disponible el limitador se abre (deja pasar) en vez de tumbar el login.

### MFA

TOTP opcional por usuario (`/auth/mfa/setup` → `/auth/mfa/enable` → `/auth/mfa/verify`). Puede hacerse obligatorio para administradores a nivel de instancia (`DIDACTA_REQUIRE_MFA_ADMIN`) o por política de organización (capability <span class="didacta-chip didacta-chip--ee">EE</span> `feat:mfa.enforcement`). Con la obligación activa y sin MFA verificado, la API responde `403 {"code": "mfa_required"}` en todo lo que no sea el propio flujo de MFA y el perfil mínimo.

### Roles

El claim `roles[]` lleva los roles de la persona: `super_admin`, `tenant_admin`, `formador`, `alumno`, `auditor`, `empresa_manager` e `inspector` (solo lectura del seguimiento Fundae, acotado a los grupos que se le concedan). `super_admin` es de la instancia y **no se puede asignar por la API**; los demás los asigna un administrador (`POST /admin/users/:id/roles`).

### Acceso de soporte

El plano de control puede abrir una ventana de acceso de soporte a un aula (ver [Instancia y plano de control](reference/instance-operations.md#acceso-de-soporte)). Se canjea con `POST /auth/support-access/redeem` y da un `accessToken` **sin refresh token** que caduca con la concesión (máximo 15 minutos). El token lleva el claim `sup`; la sesión es de un usuario de soporte propio del aula, visible en su lista de miembros, y no puede, por ejemplo, exportar datos personales de ventas.

## 2. API keys (integraciones)

Para sistemas externos. Cada key pertenece a un usuario administrador de una organización y lleva **scopes** explícitos. **Solo los administradores (`tenant_admin`, `super_admin`) crean y gestionan keys**: una key concede poderes sobre todo el aula (matricular a cualquiera, leer compras ajenas), no sobre la persona que la crea. Un `formador` o un `alumno` recibe `403 ADMIN_FORBIDDEN_ADMIN_ROLE_REQUIRED`.

```bash
# Crear (como admin) — el token en claro solo se devuelve UNA vez
curl -X POST https://tu-instancia/api/v1/auth/api-keys \
  -H "Authorization: Bearer <accessToken>" \
  -H "Content-Type: application/json" \
  -d '{"name": "CRM", "scopes": ["enrollments:write", "courses:read"], "expiresAt": "2027-12-31T00:00:00Z"}'
# → { "token": "lmsk_...", … }

# Usar — esquema ApiKey, no Bearer
curl -H "Authorization: ApiKey lmsk_..." https://tu-instancia/api/v1/inscribe/courses
```

### Scopes

Los scopes salen de un **catálogo cerrado**: pedir uno que no esté en la lista da `400`, y una key antigua con un scope fuera del catálogo lo pierde al usarse. `expiresAt` es opcional (ISO-8601); sin él la key no caduca hasta que se revoque.

| Scope | Qué permite |
| --- | --- |
| `enrollments:write` | `POST /inscribe` y `POST /inscribe/revoke`: alta y baja de matrículas. |
| `enrollments:read` | `GET /integrations/learners/state`: estado de **cualquier** email del aula. Va separado de `write` a propósito. |
| `courses:read` | `GET /inscribe/courses`, `GET /inscribe/access-groups` y `GET /integrations/courses[/:id]`: catálogo y fichas. |
| `orders:write` | `POST /integrations/orders`: guardar en el perfil del alumno una compra hecha en una tienda externa. |
| `orders:read` | `GET /integrations/learners/orders`: historial de compra de un alumno por email. |
| `community:post` | `/community-api/*`: listar espacios y publicar posts (el dueño de la key debe ser administrador). |
| `agent:courses:read`, `agent:courses:write`, `agent:live:write`, `agent:community:post` | Puerta de agentes (`/mcp`). Se crean con `POST /agent/keys` —el servidor fija los cuatro scopes—; ver [MCP](mcp.md). |

- Las keys se listan sin token (`GET /auth/api-keys`, o `GET /admin/api-keys` para las de **todo el aula**) y se revocan con `DELETE` (se marca `revokedAt`; no se borra la fila).
- Los endpoints con scopes son efectivamente **exclusivos de API keys**: un JWT de admin no porta scopes y no pasa el guard.
- Las API keys **no pasan por la cuota por usuario** del rate limit: cuentan en el cubo de la organización (ver [Convenciones](conventions.md#rate-limiting)).
- Cada alta y revocación queda en el log de auditoría (`admin.api_key.created` y `admin.api_key.revoked`).

## 3. Credencial de provisioning

Credencial de **instancia**, no de organización: la usa un plano de control externo (Didacta Cloud, o un script tuyo) para operar la lista de organizaciones sin hacerse pasar por una persona. No tiene tenant ni usuario y solo abre las rutas marcadas como operables por máquina (listar, ver, crear y cambiar estado de organizaciones, congelar altas, puerta de suscripción, accesos de soporte, dominios, consumo, vídeo y cobros de plataforma; además `GET /admin/system/health-detail`). El detalle está en [Instancia y plano de control](reference/instance-operations.md).

```bash
# La emite un super_admin. El token en claro solo se devuelve UNA vez;
# emitir otra revoca la anterior (como mucho hay una viva por instancia).
curl -X POST https://tu-instancia/api/v1/admin/provisioning-credential \
  -H "Authorization: Bearer <accessToken de super_admin>"
# → { "id": "…", "token": "didprov_…", "preview": "didprov_…", "createdAt": "…" }

# Usarla
curl -H "Authorization: Provisioning didprov_..." https://tu-instancia/api/v1/admin/tenants

# Revocarla (idempotente)
curl -X DELETE https://tu-instancia/api/v1/admin/provisioning-credential \
  -H "Authorization: Bearer <accessToken de super_admin>"
```

- De la credencial solo se guarda el SHA-256 (cifrado en reposo): si pierdes el token, emites otro.
- **401** `AUTH_PROVISIONING_CREDENTIAL_INVALID` si no vale o está revocada; **403** `AUTH_PROVISIONING_ROUTE_NOT_ALLOWED` si es válida pero la ruta no está en la lista blanca (por ejemplo, renombrar una organización exige un super_admin con sesión).
- En una instalación de un solo proceso, revocar corta en la petición siguiente; con varias réplicas, hasta 60 segundos después en las demás.
- Lo que hace queda en el audit log con `actorKind: "provisioning"` y el id de la credencial, distinguible de una acción humana.

## 4. Agentes de IA (MCP)

El aula expone un servidor MCP en `POST /api/v1/mcp` (Streamable HTTP, JSON-RPC 2.0) con claves de agente. Cómo conectar un cliente, qué herramientas hay y qué **no** puede hacer un agente está en la página [MCP](mcp.md).

## 5. SSO corporativo

- **OIDC** (`/auth/oidc/*`) y **SAML 2.0** (`/auth/saml/*`) — la *configuración* es <span class="didacta-chip didacta-chip--ee">EE</span> (capabilities `feat:sso.oidc` / `feat:sso.saml`); sin configurar, el flujo público responde 404.
- **WP-SSO** (`/modules/wp-sso/*`) — entrada desde una sesión WordPress con token corto HMAC (módulo Community).
- **SCIM 2.0** (`/scim/v2/*`) — aprovisionamiento desde Okta/Entra con Bearer estático propio, emitido en el panel (<span class="didacta-chip didacta-chip--ee">EE</span> `feat:scim`).

## Notificaciones en tiempo real (SSE)

`EventSource` no admite headers, así que los streams usan un **ticket efímero**: `POST /me/notifications/stream-ticket` (y `POST /modules/messaging/stream-ticket` para mensajería) devuelve un token de 60 segundos que se pasa como query param (`?ticket=`) al conectar el stream.
