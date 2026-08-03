# Variables de entorno

Referencia **1 a 1** de todas las variables de entorno de Didacta, agrupadas por área. La plantilla comentada vive en [`.env.example`](https://github.com/va360labs/didacta-io/blob/main/.env.example) del repo.

**Leyenda de «Componente»**: `api` = NestJS (`apps/api`) · `web` = Next.js (`apps/web`) · `worker` = workers BullMQ dentro del proceso API · `compose` = solo la lee Docker Compose · `build` = solo en build de la imagen · `tests` = suites de test/E2E.

## Resumen operativo

**Mínimo para arrancar** (la app falla o se degrada gravemente sin ellas):

1. `ADMIN_DATABASE_URL` (o `DATABASE_URL` como fallback — ver § Base de datos)
2. `REDIS_URL`
3. `AUTH_SECRET` — la única que provoca un **crash duro al arrancar** si falta o mide menos de 32 caracteres.

Con `docker-compose.alpha.yml`, las dos primeras se componen solas: **solo `AUTH_SECRET` es obligatoria**.

**Fuertemente recomendadas en producción:**

- `WEB_PUBLIC_URL` — sin ella, todos los emails llevan enlaces a `localhost`.
- `TENANT_SETTINGS_ENC_KEY` — fíjala **antes** de configurar SSO OIDC, SCIM, SMTP por tenant, Stripe o Zoom S2S.
- `METRICS_TOKEN` — si `/metrics` es alcanzable desde Internet.
- `AUDIT_REPORT_HMAC_KEY` — el fallback embebido es público (está en el repo).
- `WEB_PUBLIC_ALLOWED_HOSTS` — si hay SSO activo.
- `POSTGRES_PASSWORD` y `MINIO_ROOT_PASSWORD` distintos del default `didacta_dev`.

---

## 1. Núcleo / aplicación

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `NODE_ENV` | No | `production` | api, web, compose | Modo de ejecución. Controla el nivel del logger Pino (`info` en producción, `debug` en el resto). Con `test` se desactivan los workers BullMQ y la telemetría. `DIDACTA_DEV_BYPASS` se ignora si vale `production`. |
| `API_PORT` | No | `4000` | api, compose | Puerto de la API NestJS. En `docker-compose.alpha.yml` el puerto **interno** está fijado a 4000 y esta variable solo controla el mapeo del host. |
| `WEB_PORT` | No | `3000` | web, compose | Puerto de Next.js. Igual que `API_PORT`: en el compose alpha solo mapea el host. |
| `DIDACTA_IMAGE_TAG` | No | la versión recomendada del repo | compose | Tag de la imagen `didactaio/community` que levanta el compose. Fija siempre una versión concreta. |
| `DIDACTA_CORE_VERSION` | No | derivada del build / `DIDACTA_IMAGE_TAG` | api, compose | Fuente de verdad de la versión del core: la consumen `/healthz`, la validación de compatibilidad de módulos del marketplace y la telemetría. Normalmente no se toca. |
| `RLS_ENFORCEMENT` | No | `on` | api | Enforcement de Row-Level Security en runtime: `off` \| `warn` \| `on`. En `warn`/`on` cada query con contexto de tenant viaja con `set_config('app.current_tenant_id')`; las queries sin contexto se loguean a nivel warning (`warn`) o error (`on`). El aislamiento es real porque `DATABASE_URL` conecta con el rol `didacta_app` (sin `BYPASSRLS`) por defecto — ver § Base de datos. |
| `LOG_DB_QUERIES` | No | — | api | Con `true`, Prisma loguea todas las queries SQL. Muy verboso; solo debugging. |
| `DIDACTA_CNAME_TARGET` | No | `cname.didacta.io` | api | Target CNAME que la UI de dominios personalizados muestra al admin. |

## 2. URLs públicas y enrutado API ↔ Web

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `WEB_PUBLIC_URL` | Muy recomendada en prod | cascada (ver nota) | api, worker | URL base pública del frontend. Construye todos los enlaces absolutos de los emails (reset de contraseña, invitaciones, recordatorios, digests…). Cascada: esta variable → cabeceras `X-Forwarded-*` del request → `http://localhost:3000`. |
| `WEB_PUBLIC_ALLOWED_HOSTS` | No | — | api | Allowlist CSV de hosts para redirects que portan tokens de sesión (callbacks SSO / WP-SSO). Sin ella, el host del request nunca se usa para redirigir tokens — protege contra open-redirect. Formato: `dominio1.com,dominio2.com`. |
| `PUBLIC_API_URL` | No | cae a la base web | api | URL base pública de la API. La usan los emails con branding (assets/logo servidos desde la API) y es fallback del `redirect_uri` OIDC y de las URLs SAML. |
| `PUBLIC_BASE_URL` | No | `''` (ruta relativa) | api | Base absoluta con la que el storage local prefija `/api/v1/storage/file/…` para que las URLs de ficheros sean absolutas. |
| `WEB_BASE_URL` | No | `http://localhost:3000` | api | URL base que se inyecta en el `ModuleContext` de los módulos del marketplace. Distinta de `WEB_PUBLIC_URL` (que es la del core). |
| `API_INTERNAL_URL` | No | `http://localhost:4000` | web | URL interna de la API para los `rewrites` de Next.js, el fetch server-side y el middleware. En el navegador siempre se usa el mismo origin — sin CORS. |
| `NEXTAUTH_URL` | No | — | api | Legado de NextAuth; solo segundo fallback de la base web en OIDC/SAML. No usar en instalaciones nuevas. |

!!! note "El frontend no expone variables al navegador"
    Didacta **no usa ninguna variable `NEXT_PUBLIC_*` propia** en el bundle del cliente: toda la configuración del web es server-side y el navegador habla siempre con su mismo origin gracias a los rewrites de Next.js.

## 3. Base de datos (PostgreSQL)

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `ADMIN_DATABASE_URL` | **Sí** (o `DATABASE_URL` como fallback) | — | api, worker, compose | Conexión de **administración** (superuser/owner): `postgresql://user:pass@host:port/db?schema=public`. El entrypoint la usa SOLO para migraciones + `rls.sql` + `grants.sql` — nunca para servir tráfico. En el compose alpha se construye automáticamente desde `POSTGRES_*`. |
| `DATABASE_URL` | No — **dejar vacía** | derivada de `ADMIN_DATABASE_URL` | api, worker, compose | Conexión de **runtime** de la app. Si la dejas vacía (el caso normal), el entrypoint la deriva sustituyendo usuario/contraseña de `ADMIN_DATABASE_URL` por el rol `didacta_app` (sin `BYPASSRLS` — aislamiento RLS real). Si la defines explícitamente, se respeta tal cual (upgrade path: instalaciones existentes que solo tenían esta variable con el superuser siguen arrancando, con una degradación logueada). |
| `POSTGRES_USER` | No | `didacta` | compose | Usuario del contenedor Postgres (el de `ADMIN_DATABASE_URL`). |
| `POSTGRES_PASSWORD` | No | `didacta_dev` | compose | Contraseña del contenedor Postgres. **Cámbiala en cualquier despliegue no local** (el compose publica Postgres solo en `127.0.0.1` precisamente por este default). |
| `POSTGRES_DB` | No | `didacta` | compose | Nombre de la base de datos. |
| `POSTGRES_PORT` | No | `5432` | compose | Puerto del host mapeado a Postgres (publicado solo en loopback). |
| `POSTGRES_APP_PASSWORD` | No | autogenerada y persistida en el volumen de datos | compose | Contraseña del rol de runtime `didacta_app`. Si la dejas vacía (el caso normal), el entrypoint la genera una sola vez y la persiste — no hace falta que la sepas. Fijarla solo si querés controlarla desde tu secrets manager. |

## 4. Redis, colas y workers

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `REDIS_URL` | **Sí** (en la práctica) | — | api, worker, compose | Conexión Redis (`redis://redis:6379`). Sin ella se degradan silenciosamente todos los workers BullMQ (outbox, digests, broadcasts, purgas GDPR, recordatorios…), el rate limiting distribuido, la mensajería realtime y las notificaciones SSE. |
| `REDIS_PORT` | No | `6379` | compose | Puerto del host mapeado a Redis (solo loopback: Redis va sin autenticación). |
| `COMMUNITY_DIGEST_CRON` | No | `0 9 * * 1` | worker | Cron del digest semanal de comunidad (lunes 09:00 UTC). |
| `COMMUNITY_BROADCAST_BATCH_SIZE` | No | `5` | worker | Destinatarios por lote del worker de broadcast de comunidad. |
| `COMMUNITY_BROADCAST_INTERVAL_MS` | No | `10000` | worker | Milisegundos entre lotes de broadcast (throttling anti-spam del SMTP). |
| `LESSON_UNLOCK_CRON` | No | `*/10 * * * *` | worker | Cron del notificador de lecciones desbloqueadas (drip content). |
| `PAYMENT_SUBSCRIBERS_SYNC_CRON` | No | `*/15 * * * *` | worker | Cron de sincronización de suscriptores desde las conexiones de pago. |
| `REFERRALS_APPROVAL_CRON` | No | `30 * * * *` | worker | Cron de aprobación automática de referidos. |
| `SURVEYS_REMINDER_CRON` | No | `*/15 * * * *` | worker | Cron del recordatorio de encuestas post-clase. |
| `SURVEYS_REMINDER_DELAY_HOURS` | No | `24` | worker | Horas tras la clase antes de enviar el recordatorio de encuesta. |
| `SURVEYS_HASH_SECRET` | No | cae a `AUTH_SECRET` | api | Secreto para hashear identificadores en respuestas de encuestas (anonimización). |

## 5. Autenticación, sesiones y cifrado

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `AUTH_SECRET` | **Sí** | — | api, worker | Secreto de firma de JWT (access + refresh), tickets de inscripción y tokens de desuscripción. **Mínimo 32 caracteres**: la app lanza error al arrancar si falta o es más corta. Generar con `openssl rand -base64 32`. Cambiarla invalida todas las sesiones. |
| `AUTH_URL` | No | `https://didacta.local` | api | Issuer (`iss`) de los JWT. También base por defecto de las URLs de éxito/cancelación de checkouts de Stripe. |
| `AUTH_SIGNUP_ENABLED` | No | `false` | api, web | Habilita el registro público (`POST /auth/signup` + página `/signup`). Cerrado por diseño: las altas reales entran por inscripción o invitación de admin. Solo `true` en stacks dev/E2E. |
| `TENANT_SETTINGS_ENC_KEY` | Crítica si usas integraciones | — (cascada) | api, compose | Clave AES-256 en **hex de 64 caracteres** que cifra los secretos at-rest por tenant (client secret OIDC, token SCIM, SMTP por tenant, claves Stripe, Zoom S2S, bot de Telegram). Cascada: esta variable → fichero `${STORAGE_ROOT}/.didacta-secret-key` (autogenerado, `0600`) → clave efímera en memoria (los secretos se pierden al reiniciar). Generar con `openssl rand -hex 32`. **Jamás rotar sin plan de re-cifrado.** |
| `TENANT_SETTINGS_ENC_KEY_FILE` | No | `${STORAGE_ROOT}/.didacta-secret-key` | api | Path alternativo del fichero de la clave maestra, para separar clave y storage en volúmenes distintos. |
| `DIDACTA_REQUIRE_MFA_ADMIN` | No | `false` | api | Fuerza MFA a los admins de toda la instancia. Acepta `true`/`1`/`yes`/`on`. Independiente de la política por tenant (capability Enterprise `feat:mfa.enforcement`). |
| `OIDC_REDIRECT_URI` | No | cae a `PUBLIC_API_URL` | api | `redirect_uri` enviado al IdP OIDC; debe coincidir con el registrado en el proveedor. |
| `SAML_PUBLIC_API_URL` | No | cae a `PUBLIC_API_URL` | api | Base pública de la API para el ACS callback de SAML. |
| `SAML_SP_ENTITY_ID_BASE` | No | `<base API>/saml` | api | Base del `entityID` del Service Provider SAML. |
| `RATE_LIMIT_COMMUNITY_AUTH_PER_MIN` | No | `100` | api | Peticiones/minuto para usuarios autenticados en plan Community (Enterprise: 1000, no configurable). |
| `RATE_LIMIT_COMMUNITY_PUBLIC_PER_MIN` | No | `30` | api | Peticiones/minuto para tráfico anónimo en Community (Enterprise: 300). Ventana fija de 60 s. |

## 6. Email / SMTP

Las tres primeras funcionan en conjunto: si falta cualquiera de `SMTP_HOST`, `SMTP_PORT` o `SMTP_FROM`, no hay SMTP global — solo enviarán email los tenants con SMTP propio configurado en el panel.

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `SMTP_HOST` | No | `mailpit` en el compose | api, worker, compose | Host del servidor SMTP global. |
| `SMTP_PORT` | No | `1025` (Mailpit) | api, compose | Puerto SMTP. |
| `SMTP_FROM` | No | `Didacta <noreply@didacta.local>` | api, compose | Remitente de todos los emails. Formato RFC: `Nombre <email@dominio>`. |
| `SMTP_USER` | No | `''` | api | Usuario de autenticación SMTP. Vacío = sin auth (caso Mailpit). |
| `SMTP_PASS` | No | `''` | api | Contraseña SMTP (nombre preferente; existe el alias `SMTP_PASSWORD` por compatibilidad). |
| `SMTP_SECURE` | No | lo decide nodemailer por el puerto | api | Fuerza TLS implícito. Solo `'true'`/`'false'` literales tienen efecto. |
| `MAILPIT_SMTP_PORT` | No | `1025` | compose | Puerto del host del SMTP de Mailpit. |
| `MAILPIT_UI_PORT` | No | `8025` | compose | Puerto del host de la UI de Mailpit (solo loopback: muestra **todos** los correos, incluidos los resets de contraseña). |

## 7. Object storage (local / S3 / MinIO)

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `STORAGE_DRIVER` | No | `local` en el compose; autodetección sin definir | api, compose | `local` fuerza disco; `s3` fuerza S3 (y **falla al arrancar** si faltan `S3_ENDPOINT`/`S3_BUCKET`/`S3_ACCESS_KEY`/`S3_SECRET_KEY`). Sin definir, intenta S3 y cae a local si la config está incompleta. |
| `STORAGE_ROOT` | No | `/app/data/storage` | api, compose | Raíz del volumen persistente: uploads (logos, certificados, SCORM, evidencias Fundae) **y** la clave maestra `.didacta-secret-key`. |
| `S3_ENDPOINT` | Sí, si usas S3 | — | api, compose | Endpoint S3-compatible (`http://minio:9000`, AWS, Hetzner…). |
| `S3_BUCKET` | Sí, si usas S3 | — | api | Nombre del bucket. |
| `S3_ACCESS_KEY` | Sí, si usas S3 | — | api | Access key (alias aceptado: `S3_ACCESS_KEY_ID`). |
| `S3_SECRET_KEY` | Sí, si usas S3 | — | api | Secret key (alias aceptado: `S3_SECRET_ACCESS_KEY`). |
| `S3_REGION` | No | `us-east-1` | api | Región del bucket. |
| `S3_FORCE_PATH_STYLE` | No | `true` | api | URLs path-style (`endpoint/bucket/key`). Necesario para MinIO. |
| `S3_PRESIGNED_TTL_SECONDS` | No | default del SDK | api | TTL en segundos de las URLs prefirmadas. |
| `MINIO_ROOT_USER` | No | `didacta` | compose | Usuario root del contenedor MinIO (profile `s3`). |
| `MINIO_ROOT_PASSWORD` | No | `didacta_dev` | compose | Contraseña root de MinIO. Cámbiala fuera de local. |
| `MINIO_API_PORT` | No | `9000` | compose | Puerto del host de la API S3 de MinIO. |
| `MINIO_CONSOLE_PORT` | No | `9001` | compose | Puerto del host de la consola web de MinIO. |

## 8. IA — AI Gateway (BYOK)

La configuración canónica de IA es **por tenant**, desde el panel de administración. Estas variables definen la clave de cifrado y el proveedor global de respaldo. Los nombres del bloque `DEFAULT_AI_*` se construyen por propósito: `chat` y `embed`.

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `AI_CONFIG_ENCRYPTION_KEY` | Sí, para configurar IA por tenant | — | api | Clave AES-256-GCM que cifra las API keys de IA por tenant. **Hex de exactamente 64 caracteres** o el módulo falla al inicializar. Una sola clave por instancia. |
| `DEFAULT_AI_CHAT_PROVIDER` | No | — | api | Proveedor global por defecto para chat, usado cuando el tenant no tiene configuración propia. |
| `DEFAULT_AI_CHAT_API_KEY` | No | — | api | API key del proveedor global de chat. |
| `DEFAULT_AI_CHAT_MODEL` | No | `''` | api | Modelo por defecto para chat. |
| `DEFAULT_AI_CHAT_BASE_URL` | No | — | api | Base URL alternativa (proxies, gateways compatibles, self-hosted). |
| `DEFAULT_AI_EMBED_PROVIDER` | No | — | api | Proveedor global para embeddings. |
| `DEFAULT_AI_EMBED_API_KEY` | No | — | api | API key del proveedor de embeddings. |
| `DEFAULT_AI_EMBED_MODEL` | No | `''` | api | Modelo de embeddings. |
| `DEFAULT_AI_EMBED_BASE_URL` | No | — | api | Base URL del proveedor de embeddings. |

## 9. Zoom (clases en vivo)

Las credenciales Server-to-Server de Zoom **no son variables de entorno**: viven cifradas por tenant (por eso importa `TENANT_SETTINGS_ENC_KEY`).

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `ZOOM_WEBHOOK_SECRET` | Sí, para recibir webhooks | — | api | Secret token de Zoom para verificar la firma HMAC-SHA256 del webhook. Sin él, los eventos se rechazan. |
| `ZOOM_REMINDER_CRON` | No | `*/5 * * * *` | worker | Cron del worker de recordatorios de clase en vivo. |
| `ZOOM_REMINDER_HOURS_BEFORE` | No | `2` | worker | Horas de antelación del recordatorio. |
| `ZOOM_ATTENDANCE_SYNC_CRON` | No | `*/10 * * * *` | worker | Cron de reconciliación de asistencia contra la API de Zoom. |

## 10. Pagos — Stripe (módulos billing y subscriptions)

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `STRIPE_SECRET_KEY` | Sí, para activar pagos | — | api | Clave secreta de Stripe. Una cuenta por instancia, compartida por billing y subscriptions. Si falta, los endpoints de billing devuelven 503 y el resto de la app sigue operativa. |
| `STRIPE_WEBHOOK_SECRET` | No | cae a `SUBSCRIPTIONS_WEBHOOK_SECRET` | api | Secret del webhook de billing (`…/modules/billing/webhook`). |
| `BILLING_SUCCESS_URL_BASE` | No | `<AUTH_URL>/cursos` | api | Base de la URL de éxito del checkout de compra puntual. |
| `BILLING_CANCEL_URL_BASE` | No | `<AUTH_URL>/cursos` | api | Base de la URL de cancelación del checkout puntual. |
| `SUBSCRIPTIONS_WEBHOOK_SECRET` | No | cae a `STRIPE_WEBHOOK_SECRET` | api | Secret de un webhook **dedicado** para suscripciones (recomendado separarlo para aislar auditoría). |
| `SUBSCRIPTIONS_SUCCESS_URL_BASE` | No | `<AUTH_URL>/cuenta/suscripciones` | api | Base de la URL de éxito del checkout de suscripción. |
| `SUBSCRIPTIONS_CANCEL_URL_BASE` | No | `<AUTH_URL>/cuenta/suscripciones` | api | Base de la URL de cancelación del checkout de suscripción. |
| `SUBSCRIPTIONS_GRACE_PERIOD_DAYS` | No | `3` | api | Días desde el primer fallo de cobro hasta marcar la suscripción `UNPAID` y pausar el enrollment. |
| `SUBSCRIPTIONS_GRACE_EXPIRATION_CRON` | No | `0 * * * *` | worker | Cron que expira suscripciones con gracia vencida. |
| `SUBSCRIPTIONS_DAILY_CRON` | No | `0 9 * * *` | worker | Cron del resumen diario y avisos previos de renovación. |
| `SUBSCRIPTIONS_DAILY_TZ` | No | `UTC` | worker | Zona horaria IANA del cron diario (`Europe/Madrid`…). |
| `SUBSCRIPTIONS_RENEWAL_WINDOW_DAYS` | No | `7` | worker | Ventana en días del aviso previo a renovación/caducidad. |

## 11. Inscripción de miembros / Telegram

La configuración canónica vive **por tenant** en Administración → Configuración → Registro (verificadores exigidos, bot de Telegram con token cifrado, email del aprobador). Estas variables son solo el **fallback global** para despliegues mono-tenant; en instalaciones nuevas, déjalas vacías.

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `TELEGRAM_BOT_TOKEN` | No | — | api | Token del bot (fallback global). Requiere `TELEGRAM_GROUP_ID` a la vez. Si un tenant exige el verificador de Telegram y no hay bot, su inscripción responde 503 fail-closed. |
| `TELEGRAM_GROUP_ID` | No | — | api | ID numérico del grupo (con prefijo `-100…` en supergrupos). |
| `TELEGRAM_BOT_USERNAME` | No | — | api, web | Username del bot **sin `@`**, necesario para el Telegram Login Widget del formulario de inscripción. |
| `MEMBER_APPROVAL_EMAIL` | No | — | api | Email del aprobador que recibe las solicitudes de alta (fallback global; preferir el ajuste por tenant). |
| `MEMBER_PURGE_CRON` | No | `0 3 * * *` | worker | Cron de la purga GDPR de solicitudes caducadas. |
| `MEMBER_RETENTION_DAYS` | No | `90` | worker | Días de retención antes de purgar los datos de las solicitudes. |

## 12. Licencia Enterprise y marketplace

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `DIDACTA_LICENSE_KEY` | No | — (modo Community) | api, compose | JWT de licencia Enterprise firmado por Didacta, leído **una vez al arrancar**. Sin él: `License: community` y capabilities EE deshabilitadas. Inválido: HTTP 402/401 en endpoints gateados. Cambiar de licencia exige reiniciar el contenedor. |
| `DIDACTA_DEV_BYPASS` | No | `false` | api, compose | Bypass de licencia: activa **todas** las capabilities. **Solo desarrollo** — se ignora con `NODE_ENV=production` y emite un WARN visible. |
| `MARKETPLACE_PUBLIC_KEYS_DIR` | No | directorio embebido | api | Directorio con las claves públicas que verifican la firma de los paquetes `.zip` de módulos del marketplace. |

## 13. Telemetría, registro y observabilidad

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `DIDACTA_TELEMETRY_DISABLED` | No | `false` | api | Desactiva el latido diario anónimo. Acepta `true`/`1`/`yes`. Ver [Telemetría](telemetria.md). |
| `DIDACTA_TELEMETRY_URL` | No | cae a `DIDACTA_REGISTRY_URL` | api | Endpoint alternativo del latido (tests, proxies corporativos). Timeout fijo de 5 s. |
| `DIDACTA_REGISTRY_URL` | No | — (registro deshabilitado) | api | Endpoint del **registro opt-in identificado** (nivel distinto del latido anónimo). |
| `METRICS_TOKEN` | No | — (`/metrics` abierto) | api | Bearer token opcional del endpoint Prometheus `/metrics`. Si está definido, se exige. |
| `AUDIT_REPORT_HMAC_KEY` | No | clave de fallback embebida | api | Master key HMAC-SHA256 que firma los informes de auditoría exportados (verificables offline con `tools/audit-report-verify.mjs`). En producción **debe** sobrescribirse: el fallback es público. |
| `WEBHOOKS_TIMEOUT_MS` | No | `5000` | api, worker | Timeout del POST a un endpoint de webhook saliente. |
| `WEBHOOKS_COMMUNITY_MAX_ENDPOINTS` | No | `1` | api | Máximo de endpoints de webhook por tenant en Community (Enterprise: 20). |
| `WEBHOOKS_COMMUNITY_MAX_EVENTS` | No | `3` | api | Máximo de tipos de evento por endpoint en Community (Enterprise: ilimitado). |
| `WEBHOOKS_ENTERPRISE_MAX_ENDPOINTS` | No | `20` | api | Máximo de endpoints con la capability `feat:api.webhooks.high_throughput`. |

## 14. Seed de desarrollo (`BOOTSTRAP_*`)

El **setup wizard real** (`/setup`) no lee ninguna variable de entorno: el primer arranque se configura por la interfaz web. Estas variables son exclusivas del seed programático de desarrollo (`pnpm db:seed`).

| Nombre | Obligatoria | Default | Componente | Descripción |
|---|---|---|---|---|
| `BOOTSTRAP_TENANT_SLUG` | No | `demo` | seed | Slug del tenant sembrado. |
| `BOOTSTRAP_TENANT_NAME` | No | `Demo` | seed | Nombre visible del tenant. |
| `BOOTSTRAP_EMAIL` | No | `admin@example.com` | seed | Email del admin inicial. |
| `BOOTSTRAP_NAME` | No | `Admin` | seed | Nombre del admin inicial. |
| `BOOTSTRAP_PASSWORD` | Sí, para el seed | — | seed | Contraseña del admin. Sin default deliberadamente, para no crear credenciales conocidas. |
| `BOOTSTRAP_DOMAINS` | No | `localhost,127.0.0.1` | seed | Dominios CSV asociados al tenant (resolución de tenant por host). |

## 15. Tests y E2E (contribuidores)

| Nombre | Default | Descripción |
|---|---|---|
| `TEST_DATABASE_URL` | — | BD de los tests de integración (Postgres efímero de `docker-compose.test.yml`, puerto 5433). |
| `E2E_BASE_URL` | `http://localhost:3010` | `baseURL` de Playwright. |
| `E2E_API_URL` | `http://localhost:3000` | Base de la API para los helpers HTTP de los E2E. |
| `E2E_ADMIN_EMAIL` / `E2E_ADMIN_PASSWORD` | — | Credenciales del admin sembrado con el que los E2E hacen login. |
| `E2E_TENANT_SLUG` | `demo` | Tenant contra el que corre la suite E2E. |
| `E2E_AUTH_SECRET` | cae a `AUTH_SECRET` | Firma tickets de Telegram en los E2E de inscripción. |
| `E2E_TELEGRAM_BOT_TOKEN` | cae a `TELEGRAM_BOT_TOKEN` | Firma el payload del Telegram Login Widget en E2E. |
| `E2E_STRIPE_WEBHOOK_SECRET` / `E2E_ZOOM_WEBHOOK_SECRET` | — | Firman webhooks simulados en los E2E de pagos/Zoom. |
| `PROD_SMOKE` | — | Interruptor de los smoke tests contra producción (solo con valor exacto `1`). |
| `PROD_TEST_EMAIL` / `PROD_TEST_PASSWORD` / `PROD_TENANT_SLUG` / `PROD_COURSE_SLUG` / `PROD_EXPECTED_COURSE` | — | Parámetros del smoke de producción. |

## 16. Variables de build de la imagen

Solo relevantes si construyes tu propia imagen con `docker build`:

| Nombre | Default | Descripción |
|---|---|---|
| `DIDACTA_VERSION` | versión del repo | `ARG` del Dockerfile; se materializa en `DIDACTA_CORE_VERSION`. |
| `SKIP_TYPE_CHECK` | `0` | Con `1`, el build de Next.js salta typecheck y ESLint (los tipos se validan en CI). |
| `NEXT_TELEMETRY_DISABLED` | `1` | Telemetría propia de Next.js, desactivada en la imagen oficial. |
| `HUSKY` | `0` | Desactiva hooks de Git dentro de la imagen. |
