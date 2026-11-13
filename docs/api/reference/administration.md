# Referencia — Administración

Endpoints del back-office. Todas las rutas cuelgan de `/api/v1`. **Auth**: `admin` = `tenant_admin` o `super_admin` con Bearer JWT; algunos bloques exigen `super_admin` estricto; los marcados <span class="didacta-chip didacta-chip--ee">EE</span> requieren además una capability Enterprise (sin licencia → **402**).

## Organizaciones — `/admin/tenants` (solo super_admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/tenants` · `/:id` | Lista todos los tenants · detalle. |
| GET | `/admin/tenants/capacity` | Nº de tenants y si la licencia permite crear más. |
| POST | `/admin/tenants` | Crea tenant + primer `tenant_admin` + dominio primario y envía email de bienvenida. **402** al superar el límite Community sin `feat:multi_tenant.real`. |
| PATCH | `/admin/tenants/:id` · `/:id/status` | Renombra · cambia estado (`SUSPENDED`/`ARCHIVED` invalidan todas las sesiones de sus usuarios). |
| POST · DELETE | `/admin/tenants/:id/domains[/:hostname]` | Añade dominio · lo quita (el primario no se puede quitar). |

## Usuarios — `/admin/users` (admin, scoped al tenant del token)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/users` | Lista paginada con filtros (`search`, `status`, `role`, `externalSource`, `page`, `limit`). |
| GET | `/admin/users/:id` | Detalle con roles y sesiones recientes. |
| POST | `/admin/users` | Invita: crea usuario `PENDING` + email para definir contraseña. `{ email, name?, role, accessGroupId? }`. |
| PATCH | `/admin/users/:id/status` | `ACTIVE`/`SUSPENDED`/`DEACTIVATED` (suspender invalida sesiones; no puedes suspenderte a ti). |
| POST · PATCH | `/admin/users/:id/roles` · `/roles/remove` | Asigna / quita rol. Asignables: `tenant_admin`, `formador`, `alumno`, `auditor`, `empresa_manager` (nunca `super_admin`). |
| POST | `/admin/users/:id/resend-invite` | Reenvía la invitación. |

## Invitaciones — `/admin/invitations` (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/invitations/summary` · `/admin/invitations` | Contadores y listado con estado (filtros `invitados`/`activados`/`sin-enviar`/`sin-acceso`). |
| POST | `/admin/invitations/send-batch` | Envío por lotes en segundo plano (`size`, `emails?` para priorizar, `pauseMs?`). Idempotente: nadie recibe dos veces. |

## Módulos — `/admin/modules`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/admin/modules` | admin | Módulos disponibles con estado y dependencias (`?tenantId=` para otro tenant, solo super_admin). |
| POST | `/admin/modules/:name/enable` · `disable` | admin | Activa (idempotente) · desactiva (`?force=true` cascadea). Errores: 409 dependientes activos, 422 módulo core. |
| POST | `/admin/modules/install` | super_admin | Instala un paquete ZIP (body = ZIP crudo, `Content-Type: application/zip`). Errores tipados: 413 demasiado grande, 422 firma/lint/boot, 403 vendor no confiable, 412 core incompatible, 409 nombre reservado o ya instalado. |
| GET | `/admin/modules/installed[/:name[/routes]]` | super_admin | Módulos third-party instalados · detalle · rutas expuestas. |
| DELETE | `/admin/modules/installed/:name` | super_admin | Desinstala (desregistra rutas, borra registro y blob) → 204. |

## Claves, imágenes y métricas (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET · POST · DELETE | `/admin/api-keys[/:id]` | API keys de **todo el tenant** (crear devuelve el token una sola vez; DELETE revoca → 204). |
| GET · POST | `/admin/images/inventory` · `optimize` | Inventario de imágenes con ahorro estimado · reoptimiza hasta 50 por lote. |
| GET | `/admin/stats?range=all\|7d\|30d` | Usuarios activos, cursos, matriculaciones, certificados, tasa de finalización. |
| GET | `/admin/metrics/business` | KPIs de negocio: NPS 30d, ventas, impagos, altas/bajas, uso del tutor IA, actividad. |
| GET | `/admin/system/health-detail` | Estado consolidado de BD, Redis, storage, SMTP y outbox (diagnóstico on-call). |
| GET | `/admin/rate-limit/info` | Tier de rate limit activo y límites efectivos (informativo, nunca 402). |

## SMTP — `/admin/tenant-settings/smtp` (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET · PUT | `/admin/tenant-settings/smtp` | Config sin contraseña (flags `hasPassword`, `verifiedAt`) · alta/edición (password vacío conserva el guardado; credenciales cifradas AES-256-GCM). |
| POST | `/admin/tenant-settings/smtp/test` | Email de prueba con el SMTP del tenant; si va bien sella `verifiedAt`. El error real del MTA viaja al admin. |
| POST | `/admin/tenant-settings/smtp/test-template` | Envía el email real de una plantilla del catálogo con variables. |

## Identidad corporativa <span class="didacta-chip didacta-chip--ee">EE</span>

| Método | Ruta | Capability | Qué hace |
|---|---|---|---|
| GET · PUT · DELETE | `/admin/sso/oidc/config` | `feat:sso.oidc` | Config OIDC (el `clientSecret` se cifra y nunca vuelve en GET; enviarlo en PUT lo rota). |
| POST | `/admin/sso/oidc/test-discovery` | `feat:sso.oidc` | OIDC Discovery contra el issuer. |
| GET · PUT · DELETE | `/admin/sso/saml/config` | `feat:sso.saml` | Config SAML + URLs del SP (`entityId`, `acsUrl`, `metadataUrl`) para pegar en el IdP. |
| POST | `/admin/sso/saml/test-connection` | `feat:sso.saml` | Valida el certificado PEM y la URL (SAML no tiene discovery). |
| GET · POST · DELETE | `/admin/scim/token` | `feat:scim` | Estado · genera token nuevo (`scim_…`, **mostrado una sola vez**; reemplaza el anterior) · revoca. |
| GET · PUT | `/admin/mfa-policy` | PUT: `feat:mfa.enforcement` | Política MFA del tenant (`requiredForAll`, `gracePeriodDays` 1-90). El GET es libre para pintar el upsell. |
| GET | `/super/users` | `feat:multi_tenant.real` (solo super_admin) | Usuarios de **todos** los tenants con filtros. |
| GET · POST · PATCH · DELETE | `/admin/custom-domains[...]` | `feat:custom_domains` | Dominios personalizados: alta (genera `cnameTarget` + token), verificación, suspensión/reactivación, borrado. |
| GET · POST · DELETE | `/admin/webhooks/dead-letter[...]` | `feat:api.webhooks.high_throughput` | Dead-letter de webhooks: listar · reintentar (202) · descartar (204). |

WP-SSO es Community: `GET/PUT/DELETE /admin/sso/wp/config` (admin, sin capability).

## IA — `/admin/ai` (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/ai/providers/catalog` · `/admin/ai/providers` | Proveedores disponibles (openai, anthropic, gemini, openrouter, mistral, groq, ollama, voyage) · configs del tenant (sin claves). |
| PUT · DELETE | `/admin/ai/providers/:purpose` | Configura `chat` o `embed` (`{ provider, model?, apiKey, baseUrl?, … }`; clave cifrada) · borra (vuelve al default global). |
| GET · POST | `/admin/ai-tutor/answers[/:messageId/review]` | Revisión de respuestas del tutor: listado con filtros · marcar correcta o corregida (la corrección pasa a conocimiento validado). |
| GET · POST · PATCH · DELETE | `/admin/ai-tutor/corrections[/:id]` | Conocimiento validado: CRUD (cambiar la pregunta recalcula el embedding). |
| GET | `/admin/ai-tutor/report/monthly?mes=YYYY-MM` | Informe mensual de preguntas por tema y volumen. |
| POST | `/admin/ai-tutor/courses/:courseId/index` · `/admin/ai-tutor/reindex-all` | Re-indexa un curso · todos los publicados (backfill del RAG). |

## Plantillas, moderación y registro

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/notifications/templates/keys` · `catalog` · `/admin/notifications/templates?key=` | Claves conocidas · catálogo con variables · overrides del tenant. |
| PUT · DELETE | `/admin/notifications/templates/:key` | Override por `(channel, locale)` · borrado (sin query params borra todos los del key). |
| GET · POST | `/admin/users/:userId/restrictions` | Histórico de sanciones · sanciona (`{ scopes[], reason, expiresAt? }` — el usuario sigue entrando y leyendo, no aportando). |
| POST | `/admin/users/:userId/restrictions/:id/lift` | Levanta la sanción (sella `liftedAt`, no borra). |
| GET | `/admin/users/:userId/dossier` | Expediente completo (identidad, compras, formación, actividad, sanciones). **Cada consulta queda auditada.** |
| GET | `/admin/restrictions/scopes` · `active?userIds=csv` | Áreas sancionables · sanciones vigentes en lote (máx. 200). |
| GET · POST · DELETE | `/admin/registry/status` · `opt-in` | **super_admin.** Registro opt-in de la instalación con el equipo de Didacta (`acceptTerms: true` obligatorio) · opt-out RGPD con borrado remoto. Decisión de instancia, no de tenant. |

## Fundae (admin; el rol formador no accede)

| Método | Ruta | Qué hace |
|---|---|---|
| GET · POST · PATCH · DELETE | `/admin/fundae/companies[/:id]` | Empresas bonificadas: NIF con checksum español (inmutable tras el alta), CCC, crédito. 409 si tiene grupos activos al borrar. |
| GET · POST · DELETE | `/admin/fundae/companies/:companyId/rlpt-notices[/:id]` | Notificaciones RLPT (PDF/imagen ≤10 MiB al Evidence Vault con hash). El plazo legal de 15 días se calcula solo. |
| GET · POST · PATCH | `/admin/fundae/groups[/:id]` | Grupos bonificables (estados DRAFT→ACTIVE→CLOSED/CANCELLED). |
| POST | `/admin/fundae/groups/:id/start` · `close` · `cancel` · `finalize` | Transiciones: `start` valida RLPT y crédito; `close` debita crédito; `finalize` calcula APTO/NO_APTO (umbral 75%, con `preview`). |
| GET | `/admin/fundae/groups/:id/start-xml` · `end-xml` · `audit-zip` | XML de inicio · de fin (nominal + costes) · ZIP de auditoría completo con manifest SHA-256. |
| GET · POST · PATCH · DELETE | `/admin/fundae/groups/:id/costs[/:costId]` | Costes imputados al grupo (bloqueados con el grupo cerrado). |
| GET · POST · PATCH · DELETE | `/admin/fundae/groups/:groupId/participants[/:id]` | Participantes nominales (+ `bulk-enroll` desde el curso de la acción). Soft-delete → `REMOVED`, trazabilidad Fundae. |
