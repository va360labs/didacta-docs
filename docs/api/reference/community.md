# Referencia — Comunidad y personas

Comunidad, mensajería, gamificación y retención, recursos, encuestas, referidos, theming, inscripción de miembros, membresía, mensajes automáticos de retención y ayuda y sugerencias. Todas las rutas cuelgan de `/api/v1`.

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
| GET · POST · PATCH · DELETE | `/modules/community/spaces[/:slug]` | lectura Bearer · escritura admin | Espacios; los 4 de sistema son editables pero no eliminables (409). `accessGroupIds` al crear o editar: solo para esos niveles (vacío = todos). Los de un nivel que quien mira no tiene se listan con `locked: true`, nombre, icono y `accessLevels`, sin descripción. |
| GET | `/modules/community/members` | Bearer | Directorio paginado de miembros activos. |
| GET | `/modules/community/stats` | Bearer | Miembros y cursos activos del tenant. |
| GET | `/modules/community/sidebar` | Bearer | La barra lateral del feed en una llamada: tarjeta del aula (marca, miembros, presencia), ranking del mes y últimos comentarios. Cada bloque viene `null` o vacío si no hay datos. |
| GET · POST | `/modules/community/broadcasts` | admin | Avisos masivos con estado y reanudación por lotes. |
| POST | `/modules/community/digest/run-now` | super_admin | Fuerza el digest semanal (QA) → 202. |
| GET · POST | `/modules/community/unsubscribe` | Público (token HMAC del email) | Baja de los correos de la comunidad: `GET ?token=` desde el enlace del email (responde HTML) · `POST` de **baja en un clic** (RFC 8058) desde el gestor de correo. |

**API externa** — `/community-api` (API key con scope `community:post`, cuyo dueño debe ser admin): `GET /community-api/spaces` (dónde publicar) y `POST /community-api/posts` (publica con `source='api'`; `space` inexistente → 422 con los slugs válidos).

**Errores**: `NOT_AUTHOR` / `NOT_MODERATOR` 403 · `NESTED_REPLIES_TOO_DEEP` / `REACTION_TARGET_MISSING` 422 · `TAG_NAME_EXISTS` / `SPACE_EXISTS` / `SPACE_NOT_DELETABLE` 409 · not-found 404. Espacios por nivel: leer algo de un espacio cerrado para quien mira da 404 (como si no existiera); publicar, comentar o reaccionar en él, 403 `COMMUNITY_SPACE_LOCKED`; un grupo que no existe en `accessGroupIds`, 422 `COMMUNITY_ACCESS_GROUP_UNKNOWN`. Ver [Comunidad → Espacios por nivel](../../modules/catalog/community.md#espacios-por-nivel).

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

**Retención y celebraciones (rachas, insignias, premios y contenido por nivel):**

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/gamification/me/retention` | Bearer | Mi racha, mis insignias (y el catálogo) y mis preferencias. |
| PUT | `/modules/gamification/me/prefs` | Bearer | `{ celebrate?, showStreak? }`: no salir en las celebraciones / no enseñar mi racha. |
| GET | `/modules/gamification/me/rewards` | Bearer | Premios que he recibido y los que me quedan por conseguir. |
| GET | `/modules/gamification/me/locks` | Bearer | Contenido que se desbloquea por nivel, visto por mí (no incluye lo que tengo por compra o suscripción). |
| GET | `/modules/gamification/members/summary?ids=` · `/members/:userId/summary` | Bearer | Nivel, insignias y racha de varios miembros (tarjetas) · de uno (su perfil). |
| GET · PUT | `/modules/gamification/admin/retention/settings` | admin | Interruptores: `streaksEnabled`, `badgesEnabled`, `celebrateCourses`, `celebrateIntros`, `celebrateAnniversaries`, `celebrateLevels`. |
| GET · PUT | `/modules/gamification/admin/badges[/:key]` | admin | Catálogo de insignias · renombrar o apagar una (`{ name?, enabled? }`). |
| GET | `/modules/gamification/admin/lockable-content` | admin | Cursos (y sus secciones) que se pueden cerrar por nivel. |
| GET · PUT · DELETE | `/modules/gamification/admin/locks[/:id]` | admin | Lista · cierra un curso o una sección hasta un nivel (`{ targetKind: COURSE \| MODULE, targetId, levelId, grantPaid? }`) · quita (204). |
| GET · POST · PUT · DELETE | `/modules/gamification/admin/rewards[/:id]` | admin | Premios automáticos: `{ title, triggerKind: LEVEL \| POINTS, levelId?, points?, kind: MESSAGE \| CODE, message?, url?, code?, active? }`. |
| GET | `/modules/gamification/admin/reward-grants` | admin | Registro de premios entregados. |
| POST | `/modules/gamification/admin/celebrations/anniversaries/run` | admin | Publica ya los aniversarios de hoy de esta aula (lo hace solo cada mañana). Idempotente. |

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

**Miembro:**

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/referrals/me` | Bearer | Mi código y enlace (`/join?ref=`), creándolo si no existe. 404 si el programa está inactivo; 403 si exige membresía activa y no la tengo. |
| POST | `/modules/referrals/me/terms` | Bearer | Declaro ser mayor de 18 años y acepto la versión vigente de las condiciones. Sin esto no hay enlace ni se atribuyen comisiones; 403 `REFERRALS_TERMS_REQUIRED` si la versión no es la vigente. |
| GET · PUT | `/modules/referrals/me/payee` | Bearer | Datos para cobrar en dinero (nombre completo, país, NIF/NIE o identificador fiscal, IBAN), cifrados; el GET los devuelve **enmascarados** (solo los 4 últimos) o `null`. PUT: 422 `REFERRALS_PAYEE_INVALID` con los campos que fallan (nunca sus valores). |
| POST | `/modules/referrals/me/payouts` | Bearer | Pido el cobro en dinero de lo confirmado. Necesita datos de cobro y llegar al mínimo del aula; 409 si ya hay uno pedido o faltan datos. |
| GET | `/modules/referrals/me/payouts/:id/receipt` | Bearer | Justificante en PDF de un cobro en dinero pagado propio. |
| GET | `/modules/referrals/me/stats` | Bearer | Clics, altas atribuidas, comisiones por estado, historial y liquidaciones. Funciona aunque el programa esté inactivo (`programActive=false`) para mostrar el histórico. |
| GET | `/modules/referrals/me/invitees` | Bearer | Mis invitados: iniciales, fecha de entrada, estado (`trial`, `paying`, `bought_course`, `cancelled`, `joined`) y comisión generada no anulada. **Nunca** nombre ni correo. |
| POST | `/modules/referrals/me/redeem-credit` | Bearer | Canjea lo ganado (comisiones aprobadas sin cobrar) como descuento en la próxima cuota de la membresía (abono en el saldo de cliente de Stripe). Idempotente. 403 sin membresía activa o con la opción apagada; 409 si no hay nada que canjear. |

**Público:**

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/modules/referrals/track` | Público | Registra un clic de `/join?ref=CODE` (aula por dominio; dedupe por código+día+hash de IP; la IP nunca se guarda en claro). Best-effort: nunca falla por código inválido o programa inactivo. |
| GET | `/modules/referrals/public/terms` | Público | Condiciones públicas del programa del aula: versión vigente y cifras reales (porcentaje, días de atribución y de garantía, mínimo de cobro, tope del descuento). |

**Administración (admin):**

| Método | Ruta | Qué hace |
|---|---|---|
| GET · PUT | `/modules/referrals/admin/config` | Política del programa: `commissionBps`, ámbito, ventanas, garantía, mínimo de liquidación. Los cambios **no recalculan** comisiones ya devengadas (los puntos básicos quedan sellados por comisión). |
| GET | `/modules/referrals/admin/commissions` · `referrers` | Comisiones con filtros y totales por estado · referidores con métricas. |
| POST | `/modules/referrals/admin/commissions/:id/approve` · `revoke` | Aprueba una comisión `PENDING` sin esperar la garantía · revoca (`PENDING`/`APPROVED`) con motivo obligatorio. |
| GET | `/modules/referrals/admin/cash-payouts` | Cobros en dinero del aula (pedidos y pagados) con lo necesario para pagar: nombre, identificador fiscal e IBAN. |
| POST | `/modules/referrals/admin/cash-payouts/:id/paid` | Marca pagado un cobro pedido, con la referencia de la transferencia: la liquidación pasa a `DONE` y sus comisiones a `PAID`. |
| GET | `/modules/referrals/admin/cash-payouts/:id/receipt` | Justificante en PDF de un cobro en dinero pagado del aula. |
| POST | `/modules/referrals/admin/payouts` | Liquidación manual atómica de un lote `APPROVED` con referencia externa obligatoria. |

## Theming — `/modules/theming`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET · PUT | `/modules/theming/me` | lectura Bearer · escritura admin | Theme del tenant (hue/saturación, fuentes whitelisted, titulares de acceso). `customCss`/`footerHtml` no vacíos requieren `feat:white_label` (402). |
| POST | `/modules/theming/me/reset` | admin | Vuelve a los defaults. |
| POST · DELETE | `/modules/theming/me/logo` | admin | Sube (base64, ≤2 MB, png/jpeg/svg/webp) · elimina el logo. |
| GET | `/modules/theming/tenants/:tenantId/logo` | Público | Sirve el logo (necesario en `/signin` antes de autenticar). |
| POST · DELETE | `/modules/theming/me/favicon` | admin | Sube (base64, ≤1 MB, png/jpeg/svg/webp) · quita el favicon. |
| GET | `/modules/theming/tenants/:tenantId/favicon` | Público | Sirve el favicon (también se pinta en pantallas sin sesión). |
| POST · DELETE | `/modules/theming/me/cover` | admin | Sube (≤10 MB) · quita la portada de la academia —la cabecera de su tarjeta en la comunidad—; sin ella la tarjeta vuelve a la banda de color. |
| GET | `/modules/theming/tenants/:tenantId/cover` | Público | Sirve la portada (un `<img>` no manda el token). |

## Inscripción de miembros — `/modules/member-registration`

**Flujo público** (tenant por dominio; los pasos se encadenan con tickets firmados):

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/modules/member-registration/config` | Qué pasos exige el wizard (`verifiers`, `botUsername`) y si se entra al momento (`autoApprove`: nivel gratis con correo verificado). |
| POST | `/modules/member-registration/telegram/verify` | Valida la firma del Telegram Login Widget y la pertenencia al grupo → ticket (15 min). |
| POST | `/modules/member-registration/otp/request` · `otp/verify` | Envía código al email · lo valida → `verificationToken` (30 min). |
| POST | `/modules/member-registration/register` | Crea el usuario `PENDING` y avisa al aprobador → `{ status: 'PENDING' }`. Con la [aprobación automática del nivel gratis](../../modules/catalog/member-registration.md#aprobacion-automatica-con-el-nivel-gratis), la aprueba al momento sin avisar al aprobador → `{ status: 'ACTIVE' }`. |
| GET · POST | `/modules/member-registration/decision` | `GET ?token=`: el enlace aprobar/rechazar del email del aprobador **solo lee** y redirige (302) a la pantalla de confirmación · `GET decision/preview`: qué haría ese token sin ejecutarlo · `POST`: ejecuta la decisión ya confirmada. |

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
| GET | `/membership/page` | Público | Datos de `/join`: planes activos, cursos con precio de referencia, testimonial. Con niveles, además `tiers` (lo que incluye cada uno: cursos, espacios y próximos directos), `freeSignupPath` y `freeSignupInstant`. |
| GET | `/membership/tiers` | Público | Niveles encendidos: `{ tiers: [{ id, name, accessGroupId, isFree }] }`. |
| POST | `/membership/checkout` | Público | Checkout de suscripción anónimo: `{ planId, email?, referralCode? }` → `{ url, sessionId }`. |
| GET · POST · PATCH · DELETE | `/membership/admin/plans[/:id]` | admin | Planes: nombre, periodicidad (1-12 meses), precio en céntimos, precio tachado, trial, destacado y `tierId` (el nivel; obligatorio si hay niveles de pago, se fija al crear). Borrar un plan con ventas lo desactiva. |
| GET · POST | `/membership/admin/tiers` | admin | [Niveles](../../modules/catalog/subscriptions.md#niveles): lista (cursos, precios, gente dentro, planes sin nivel y estado del registro gratis) · crea `{ name, description?, isFree?, allCourses?, courseIds? }` (el primer nivel de pago adopta los planes de siempre; el gratis abre el registro gratis). |
| PATCH · DELETE | `/membership/admin/tiers/:id` | admin | Edita `{ name?, description?, active?, courseIds?, manualReview? }` · borra un nivel sin nadie dentro (con alumnos, 409: se apaga). |
| PUT | `/membership/admin/tiers/order` | admin | Ordena los niveles (`{ tierIds }`): el orden de las tarjetas de `/join`. |
| GET · PUT | `/membership/admin/config` | admin | Página pública: activo, titulares, grupo de acceso, límite de lecciones en trial, precios por curso, testimonial. |
| GET | `/membership/checkout-session/:sessionId` | Público | Vuelta del pago de la membresía (pantalla de gracias): si la sesión de **esta** academia está completada, qué plan, cuánto y cuándo. |
| GET · PUT | `/membership/admin/retention` | admin | «Cuando alguien quiere irse»: qué se ofrece al darse de baja (pausa y meses máximos, descuento con % y meses). |
| GET | `/membership/admin/retention/summary` | admin | Resumen de bajas: cuántos pulsaron «Darme de baja», cuántos se quedaron (pausa o descuento) y los motivos. |

**Errores**: `MEMBERSHIP_PAGE_INACTIVE` 404 · `MEMBERSHIP_CONFIG_INCOMPLETE` 422 · `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING` 503 · `SUBSCRIPTIONS_STRIPE_API_ERROR` 502. Niveles: `MEMBERSHIP_TIER_NOT_FOUND` 404 · `MEMBERSHIP_TIER_INVALID` / `MEMBERSHIP_TIER_IS_FREE` 400 · `MEMBERSHIP_TIER_IN_USE` / `MEMBERSHIP_PLAN_NAME_TAKEN` 409.

## Retención — `/modules/retention` (admin)

Mensajes automáticos del aula (bienvenida, inactividad, curso completado…) y lista de miembros en riesgo. Todo exige administrar el aula; con el módulo apagado para el tenant, **403** `TENANT_MODULES_MODULE_NOT_ACTIVE`.

| Método | Ruta | Qué hace |
|---|---|---|
| GET · PUT | `/modules/retention/admin/settings` | Quién firma los mensajes (`senderUserId`) y las reglas anti-spam (`manualCooldownHours`, `sendWindowStartHour`, `sendWindowEndHour`). |
| GET | `/modules/retention/admin/templates` | Plantillas listas, en el idioma del aula. |
| POST | `/modules/retention/admin/templates/:key/activate` | Activa una plantilla con un clic (o la reenciende si ya existe). |
| GET · POST | `/modules/retention/admin/automations` | Mensajes automáticos con sus métricas · crea uno: `{ trigger, name?, days?, resendAfterDays?, body, emailEnabled?, emailSubject?, enabled? }`. |
| PATCH · DELETE | `/modules/retention/admin/automations/:id` | Edita, enciende o apaga · borra (204; se conservan sus métricas). |
| POST | `/modules/retention/admin/preview` | Vista previa con los datos reales de una persona o de quien mira: `{ trigger, body, subject?, userId? }`. |
| POST | `/modules/retention/admin/run` | Pasa el barrido de **esta** aula ahora, sin esperar a los 5 minutos (mismas reglas anti-spam). |
| GET | `/modules/retention/admin/at-risk` | Miembros en riesgo (`?reason=inactive\|paid_not_started\|stalled`, `days` 1–365, `search`): sin entrar, pagaron y no empezaron, a medias. |
| POST | `/modules/retention/admin/at-risk/:userId/recovery` | Añade a la persona al mensaje de recuperación del aula. |

Disparadores (`trigger`): `WELCOME`, `DAYS_AFTER_JOIN`, `INACTIVE`, `COURSE_COMPLETED`, `FIRST_POST`, `LEVEL_UP` y `ANNIVERSARY`.

## Ayuda y sugerencias — `/modules/soporte`

Tickets de «algo no funciona / tengo un problema / tengo una sugerencia». **El destino lo decide el servidor por los roles de la sesión**, nunca el cuerpo: un miembro escribe al **equipo del aula** (destino `AULA`); quien administra el aula escribe a **Didacta** (destino `PLATAFORMA`, con aviso al correo del operador que define `SUPPORT_OPERATOR_ALERT_EMAIL`). Con el módulo apagado, 403 `TENANT_MODULES_MODULE_NOT_ACTIVE`.

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/modules/soporte/tickets` | Bearer | Abre un ticket: `{ tipo: FALLO \| PROBLEMA \| MEJORA, texto (5–4000), contexto?, captura? }`. La captura es PNG, JPEG o WebP de hasta 5 MB en base64. **Máximo 5 tickets en 24 horas.** Responde en cuanto se guarda; los avisos salen después. |
| GET | `/modules/soporte/canal` | Bearer | A quién iría un ticket de quien llama (`AULA` o `PLATAFORMA`) y si hay alguien al otro lado; la web no enseña el botón si no. |
| GET | `/modules/soporte/tickets/mios` | Bearer | Mis tickets y su estado, del más nuevo al más antiguo. |
| GET | `/modules/soporte/tickets/:id/captura` | autor o admin | La captura de un ticket (404 a cualquier otro). |
| GET | `/modules/soporte/admin/tickets` | admin | Bandeja del equipo del aula: los 200 más nuevos, con filtros `?estado=` y `?tipo=`, y cuántos hay en cada estado. |
| GET | `/modules/soporte/admin/menu` | Bearer | `{ allowed, badge }`: si quien llama ve la bandeja y cuántos tickets nuevos hay sin coger. |
| GET · PATCH | `/modules/soporte/admin/tickets/:id` | admin | Un ticket con su autor, contexto y nota interna · cambia `estado` (`NUEVO`, `EN_CURSO`, `RESUELTO`, `DESCARTADO`) o `notaInterna` (al menos uno); si el estado cambia se avisa a quien lo abrió. |
