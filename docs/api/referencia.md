# Referencia de endpoints

Mapa completo de la superficie de la API, agrupado por bloque. Para el detalle exhaustivo de cada endpoint (schemas de request/response, códigos), usa el **Swagger de tu instancia**: `/api/docs`.

Todos los paths llevan el prefijo `/api/v1` salvo los marcados con ⚠.

## Autenticación e identidad

| Prefijo | Endpoints | Qué cubre |
| --- | --- | --- |
| `/auth` | `tenant-context`, `signup`, `signin`, `refresh`, `forgot-password`, `reset-password` | Login y ciclo de vida de tokens. |
| `/auth/mfa` | `setup`, `enable`, `verify` | TOTP + códigos de recuperación. |
| `/auth/api-keys` | crear / listar / revocar | API keys con scopes. |
| `/me` | perfil, onboarding, preferencias de notificación, cambio de contraseña, sesiones activas | El usuario autenticado. |
| `/me/modules` | GET | Módulos activos del tenant + capabilities Enterprise de la instancia (alimenta el menú). |
| `/me/notifications` | listar, marcar, stream SSE (con ticket) | Notificaciones in-app. |
| `/users` | perfil público | Perfiles visibles de otros usuarios. |
| `/auth/oidc` · `/auth/saml` | authorize/callback · metadata/login/ACS | Flujos SSO corporativo. |
| `/modules/wp-sso` | callback | SSO desde WordPress. |
| ⚠ `/scim/v2` | Users CRUD + discovery | Aprovisionamiento SCIM 2.0 (Enterprise). |

## Setup y plataforma

| Prefijo | Qué cubre |
| --- | --- |
| `/setup` | `status`, `available-modules`, `init` — [primer arranque](../instalacion/setup-wizard.md); `init` responde 409 una vez inicializado. |
| ⚠ `/api/license` | Estado público de la licencia: status, capabilities, avisos. Sin secretos. |
| `/branding` | Opciones públicas de marca del tenant. |
| `/branding/white-label` | Preview y configuración white-label (**402** sin `feat:white_label`). |
| `/system` | Comprobación de versión nueva (proxy a los tags de Docker Hub). |
| ⚠ `/healthz` `/readyz` `/livez` | Probes de salud. |
| ⚠ `/metrics` | Prometheus (Bearer opcional con `METRICS_TOKEN`). |

## Administración

| Prefijo | Qué cubre |
| --- | --- |
| `/admin/tenants` | Organizaciones: listar, crear (402 sin `feat:multi_tenant.real`), estados, dominios. |
| `/admin/users` | Usuarios: listar/paginar, invitar, roles, estados (suspender invalida sesiones). |
| `/admin/invitations` | Invitaciones pendientes y reenvío. |
| `/admin/modules` | [Activación de módulos](../modulos/gestion.md) por tenant + instalación de paquetes ZIP (`install`, `installed`, rutas). |
| `/admin/api-keys` | API keys a nivel administración. |
| `/admin/tenant-settings/smtp` | SMTP por organización. |
| `/admin/images` | Subida de imágenes de marca. |
| `/admin/stats` · `/admin/metrics/business` | Métricas de uso y de negocio. |
| `/admin/custom-domains` | Dominios personalizados (**Enterprise**). |
| `/admin/sso/oidc` · `/admin/sso/saml` | Configuración SSO (**Enterprise**). |
| `/admin/sso/wp` | Configuración WP-SSO (Community). |
| `/admin/scim/token` | Token SCIM (**Enterprise**). |
| `/admin/mfa-policy` | Política MFA de la organización (**Enterprise**). |
| `/admin/webhooks` | Dead-letter de webhooks: listar, reintentar, descartar (**Enterprise**). |
| `/admin/ai/providers` | Configuración BYOK de proveedores IA por tenant. |
| `/admin/ai-tutor` | Revisión de conversaciones del tutor IA. |
| `/admin/notifications/templates` | Plantillas de notificación. |
| `/admin/users/:id/restrictions` · `/admin/restrictions` | Moderación: restricciones de personas y dossier. |
| `/admin/registry` | Registro opt-in de la instalación. |
| `/admin/rate-limit` | Límites efectivos del tenant. |
| `/admin/fundae/*` | Empresas, grupos y participantes Fundae. |

## Integración externa

| Prefijo | Qué cubre |
| --- | --- |
| `/inscribe` | **API para integradores** (API key + scopes): alta idempotente de usuario + matrícula (`POST /inscribe`), baja (`POST /inscribe/revoke`), catálogo (`GET /inscribe/courses`, `GET /inscribe/access-groups`). |
| `/webhooks` | Webhooks salientes: info del plan, CRUD de endpoints. Ver [Convenciones](convenciones.md#webhooks-salientes). |
| `/webhooks/zoom` | Webhook **entrante** de Zoom (verificación HMAC). |

## Módulos funcionales (`/modules/<slug>`)

Cada módulo activo expone su API bajo su namespace. Los más grandes:

| Prefijo | Qué cubre |
| --- | --- |
| `/modules/courses` | Cursos, módulos y lecciones (catálogo + editor). |
| `/modules/learning` | Matrícula, progreso, finalización; `/modules/learning/paths` para itinerarios. |
| `/modules/assessments` | Quizzes y exámenes + intentos. |
| `/modules/certificates` | Plantillas y emisión de certificados (+ verificación pública). |
| `/modules/access-groups` | Grupos de acceso. |
| `/modules/community` | Posts, comentarios, reacciones, espacios (+ vista pública). |
| `/modules/messaging` | Salas, directos y canal de profesores (SSE). |
| `/modules/gamification` | Puntos, niveles y retos. |
| `/modules/billing` | Checkout de pago único + webhook Stripe. |
| `/modules/subscriptions` · `/membership` | Suscripciones recurrentes + webhook Stripe + membresía. |
| `/modules/payment-connections` | Conexiones Stripe de solo lectura + webhook WooCommerce. |
| `/modules/zoom-live` | Sesiones en directo + calendario. |
| `/modules/fundae` | Acciones formativas, exports XML/ZIP. |
| `/modules/member-registration` | Inscripción pública + administración de solicitudes. |
| `/modules/referrals` · `/modules/resources` · `/modules/surveys` · `/modules/theming` | Referidos, biblioteca de recursos, encuestas, personalización visual. |
| `/modules/ai-content` · `/modules/ai-grader` · `/modules/ai-tutor` | Los tres módulos de IA. |
| `/modules/events` | Consulta de eventos de dominio. |

Con el módulo desactivado para el tenant, todo su namespace responde 403.

## Transversales

| Prefijo | Qué cubre |
| --- | --- |
| `/storage` · `/storage/file` | Subida y descarga de ficheros (URLs firmadas). |
| `/tenant-settings` | Ajustes por organización. |
| `/audit` | Consulta del log de auditoría (+ export firmado, **Enterprise**). |
| `/formador/stats` | Métricas del formador. |

---

**~100 controllers y ~540 endpoints** en total. Este mapa cubre los prefijos; el contrato fino de cada endpoint vive en el Swagger de tu instancia (`/api/docs`), generado desde el propio código.
