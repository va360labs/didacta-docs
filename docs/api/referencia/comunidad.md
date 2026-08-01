# Referencia — Comunidad y personas

Comunidad, mensajería, gamificación, recursos, encuestas, referidos, theming, inscripción de miembros y membresía. Todas las rutas cuelgan de `/api/v1`.

**Auth**: `Bearer` = usuario autenticado · `admin` = tenant_admin/super_admin · `staff` = admin + formador · `Público` = sin sesión (tenant resuelto por el dominio).

## Comunidad — `/modules/community`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/modules/community/posts` | Bearer | Crea post: `title`, `body`, `courseId?`, `tags?` (≤10). `notifyAll` (solo admin) genera además broadcast email + campana; `important` ignora el opt-out. |
| GET | `/modules/community/posts` | Bearer | Feed con filtros (`courseId`, `authorId`, `tag`, `source`, `sort: recent\|oldest\|most_commented`, `limit`). |
| GET · PATCH · DELETE | `/modules/community/posts/:id` | Bearer | Detalle con comentarios y reacciones · edición (autor o admin) · soft-delete (autor). |
| POST | `/modules/community/posts/:id/comments` | Bearer | Comenta (1 nivel de respuesta anidada). |
| DELETE | `/modules/community/comments/:id` | Bearer (autor) | Soft-delete del comentario. |
| POST · DELETE | `/modules/community/reactions[/:id]` | Bearer | Reacción por emoji sobre post o comentario (idempotente) · retirada. |
| GET | `/modules/community/attachments` | Bearer | Adjuntos extraídos de los posts (galería). |
| GET | `/modules/community/users/search?prefix=` | Bearer | Autocomplete de menciones (máx. 8). |
| GET | `/modules/community/mentions/me` | Bearer | Mis últimas menciones. |
| GET · PUT | `/modules/community/me/preferences` | Bearer | Preferencias (p. ej. `digestOptOut`). |
| POST | `/modules/community/posts/:id/moderate` · `comments/:id/moderate` | admin | Oculta/restaura (`{ hidden, reason? }`) — reversible, distinto del borrado del autor. |
| POST | `/modules/community/posts/:id/pin` · `unpin` | admin | Fija/desfija el post en el feed. |
| GET · POST · PUT · DELETE | `/modules/community/tags[/:id]` | lectura Bearer · escritura admin | Tags curados (`name`, `color` hex, `icon`). |
| GET · POST · PATCH · DELETE | `/modules/community/spaces[/:slug]` | lectura Bearer · escritura admin | Espacios; los 4 de sistema son editables pero no eliminables (409). |
| GET | `/modules/community/members` | Bearer | Directorio paginado de miembros activos. |
| GET | `/modules/community/stats` | Bearer | Miembros y cursos activos del tenant. |
| GET · POST | `/modules/community/broadcasts` | admin | Avisos masivos con estado y reanudación por lotes. |
| POST | `/modules/community/digest/run-now` | super_admin | Fuerza el digest semanal (QA) → 202. |
| GET | `/modules/community/unsubscribe?token=` | Público (token HMAC del email) | Baja de avisos masivos; responde HTML. |

**API externa** — `/community-api` (API key con scope `community:post`, cuyo dueño debe ser admin): `GET /community-api/spaces` (dónde publicar) y `POST /community-api/posts` (publica con `source='api'`; `space` inexistente → 422 con los slugs válidos).

**Errores**: `NOT_AUTHOR` / `NOT_MODERATOR` 403 · `NESTED_REPLIES_TOO_DEEP` / `REACTION_TARGET_MISSING` 422 · `TAG_NAME_EXISTS` / `SPACE_EXISTS` / `SPACE_NOT_DELETABLE` 409 · not-found 404.

## Mensajería — `/modules/messaging`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/messaging/conversations` | Bearer | Bandeja: salas de espacio, canal de profesores y directos, con no-leídos. |
| POST | `/modules/messaging/dm` | Bearer | Abre (o crea, idempotente por par) el directo con otro miembro. |
| POST | `/modules/messaging/spaces/:slug/open` · `faculty/open` | Bearer | Abre la sala del espacio · el canal privado con profesores (auto-provisionado). |
| GET · POST | `/modules/messaging/conversations/:id/messages` | Bearer | Histórico paginado por cursor (50) · envío (`body` 1-4000, cupo 20/min). |
| POST | `/modules/messaging/conversations/:id/typing` | Bearer | Señal «escribiendo» (solo SSE, cupo 30/min) → 204. |
| POST | `/modules/messaging/conversations/:id/read` | Bearer | Marca como leída (`lastReadAt`). |
| GET | `/modules/messaging/presence` · `members?search=` | Bearer | Presencia en vivo · buscador de miembros para abrir directo. |
| POST | `/modules/messaging/stream-ticket` | Bearer | Ticket SSE de ~60 s. |
| GET | `/modules/messaging/stream?ticket=` | Ticket | **SSE**: `message.created`, `typing`, `ping`. |

**Errores**: `MESSAGING_NOT_PARTICIPANT` 403 · `MESSAGING_SELF_DM` / `MESSAGING_STAFF_NO_FACULTY` 422 · `MESSAGING_RATE_LIMITED` 429 · cuenta no operativa 403.

## Gamificación — `/modules/gamification`

**Miembro (Bearer):** `GET leaderboard?range=week|month|all` · `GET me` · `GET me/history` · `GET levels` · `GET challenges` · `GET me/perks` · `POST perks/:id/request` · `POST challenges/:id/submit` (`proofUrl?`, `note?`).

**Operador:**

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET · PUT | `/modules/gamification/admin/rules[/:key]` | admin | Reglas automáticas: puntos, techo diario, activación. |
| POST · PUT · DELETE | `/modules/gamification/admin/levels[/:id]` | admin | Niveles (crear/editar recoloca perfiles). |
| GET · POST · PUT · DELETE | `/modules/gamification/admin/perks[/:id]` | admin | Beneficios de nivel (cupo por alumno, espera). |
| GET · POST | `/modules/gamification/admin/perk-requests[/:id/handle]` | staff | Solicitudes de beneficio · atender (`APPROVED\|DONE\|REJECTED`). |
| GET · POST · PUT · DELETE | `/modules/gamification/admin/challenges[/:id]` | staff | Retos con premio y ventana de fechas (borrar no retira puntos ya dados). |
| GET · POST | `/modules/gamification/admin/submissions[/:id/review]` | staff | Entregas · aprobar (acredita puntos) o rechazar. |
| POST | `/modules/gamification/admin/backfill` | admin | Rellena el ledger con la actividad histórica (idempotente). |

**Errores**: `GAMIFICATION_CHALLENGE_CLOSED` / `GAMIFICATION_PERK_UNAVAILABLE` / `GAMIFICATION_ALREADY_SUBMITTED` / `GAMIFICATION_ALREADY_REVIEWED` 409 · validación 422 · not-found 404.

## Recursos — `/modules/resources`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET · POST | `/modules/resources/collections` | lectura Bearer · escritura staff | Colecciones (siembra las 6 por defecto) · alta con portada. |
| GET · PUT · DELETE | `/modules/resources/collections/:id` | staff (lectura Bearer) | Colección + recursos con buscador · edición · borrado **solo si está vacía**. |
| POST | `/modules/resources` | Bearer | Comparte recurso: `collectionId`, `kind: FILE\|LINK`, `title`, `url`. |
| POST | `/modules/resources/:id/download` | Bearer | Registra la descarga y devuelve la URL. |
| DELETE | `/modules/resources/:id` | autor o staff | Elimina el recurso. |

## Encuestas — `/modules/surveys` (respuestas anónimas)

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/surveys/sessions/:sessionId` | Bearer | Encuesta de una clase en directo + si ya respondí. |
| POST | `/modules/surveys/:id/responses` | Bearer | Respuesta anónima (1 por encuesta; dedupe por hash HMAC, el `userId` nunca se persiste). |
| GET | `/modules/surveys/admin[/:id/results]` | admin | Listado · resultados agregados (NPS, medias, textos). |
| POST | `/modules/surveys/admin/sessions/:sessionId` | admin | Crea la encuesta post-clase sin esperar al webhook de Zoom. |
| POST | `/modules/surveys/admin/:id/close` · `reminders/run` | admin | Cierra la encuesta · fuerza el barrido de recordatorios. |

**Errores**: `SURVEYS_CLOSED` / `SURVEYS_ALREADY_RESPONDED` 409 · `SURVEYS_INVALID_ANSWER` 422.

## Referidos — `/modules/referrals`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/referrals/me` · `me/stats` | Bearer | Mi código y enlace (`/unete?ref=`) · clics, altas, comisiones e historial. |
| POST | `/modules/referrals/track` | Público | Registra un clic (dedupe por código+día+hash de IP; la IP nunca se guarda en claro). |
| GET · PUT | `/modules/referrals/admin/config` | admin | Política del programa: `commissionBps`, ámbito, ventanas, garantía, mínimo de liquidación. |
| GET | `/modules/referrals/admin/commissions` · `referrers` | admin | Comisiones con filtros y totales · referidores con métricas. |
| POST | `/modules/referrals/admin/commissions/:id/approve` · `revoke` | admin | Aprueba · revoca con motivo obligatorio. |
| POST | `/modules/referrals/admin/payouts` | admin | Liquidación manual atómica de un lote `APPROVED` con referencia externa. |

## Theming — `/modules/theming`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET · PUT | `/modules/theming/me` | lectura Bearer · escritura admin | Theme del tenant (hue/saturación, fuentes whitelisted, titulares de acceso). `customCss`/`footerHtml` no vacíos requieren `feat:white_label` (402). |
| POST | `/modules/theming/me/reset` | admin | Vuelve a los defaults. |
| POST · DELETE | `/modules/theming/me/logo` | admin | Sube (base64, ≤2 MB, png/jpeg/svg/webp) · elimina el logo. |
| GET | `/modules/theming/tenants/:tenantId/logo` | Público | Sirve el logo (necesario en `/signin` antes de autenticar). |

## Inscripción de miembros — `/modules/member-registration`

**Flujo público** (tenant por dominio; los pasos se encadenan con tickets firmados):

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/modules/member-registration/config` | Qué pasos exige el wizard (`verifiers`, `botUsername`). |
| POST | `/modules/member-registration/telegram/verify` | Valida la firma del Telegram Login Widget y la pertenencia al grupo → ticket (15 min). |
| POST | `/modules/member-registration/otp/request` · `otp/verify` | Envía código al email · lo valida → `verificationToken` (30 min). |
| POST | `/modules/member-registration/register` | Crea el usuario `PENDING` y avisa al aprobador → `{ status: 'PENDING' }`. |
| GET | `/modules/member-registration/decision?token=` | Enlace aprobar/rechazar del email del aprobador (302 al resultado). |

Si el tenant exige un verificador no operativo (p. ej. Telegram sin bot), responde **503 fail-closed**.

**Administración** (admin):

| Método | Ruta | Qué hace |
|---|---|---|
| GET · POST | `/modules/member-registration/admin/requests` | Solicitudes pendientes con lookup de pagos · alta manual sin OTP. |
| POST | `…/admin/requests/:userId/rerun` · `decision` | Re-lanza el lookup de suscripción · aprueba/rechaza desde el panel. |
| GET · POST | `…/admin/requests/:userId/renewal-context` · `renewal-email` | Contexto de renovación (enlace Stripe) · envía el recordatorio de pago. |
| GET · POST · DELETE | `/modules/member-registration/payment-flags[/:id]` | Flags de impago (match por email o Telegram) + `POST …/import` para carga CSV atómica (≤5000). |

## Membresía — `/membership`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/membership/page` | Público | Datos de `/unete`: planes activos, cursos con precio de referencia, testimonial. |
| POST | `/membership/checkout` | Público | Checkout de suscripción anónimo: `{ planId, email?, referralCode? }` → `{ url, sessionId }`. |
| GET · POST · PATCH · DELETE | `/membership/admin/plans[/:id]` | admin | Planes: nombre, periodicidad (1-12 meses), precio en céntimos, precio tachado, trial, destacado. Borrar un plan con ventas lo desactiva. |
| GET · PUT | `/membership/admin/config` | admin | Página pública: activo, titulares, grupo de acceso, límite de lecciones en trial, precios por curso, testimonial. |

**Errores**: `MEMBERSHIP_PAGE_INACTIVE` 404 · `MEMBERSHIP_CONFIG_INCOMPLETE` 422 · `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING` 503 · `SUBSCRIPTIONS_STRIPE_API_ERROR` 502.
