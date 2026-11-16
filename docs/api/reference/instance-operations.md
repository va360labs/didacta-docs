# Referencia — Instancia y plano de control

Lo que opera **la instalación entera**, no un aula: la lista de organizaciones, la credencial de provisioning, los accesos de soporte, la puerta de suscripción, la licencia y los ajustes de instancia. Es la superficie que usa un **plano de control** externo —Didacta Cloud, o un script tuyo si autoalojas— y un `super_admin` desde el panel. Todas las rutas cuelgan de `/api/v1`.

**Leyenda de Auth**: `super_admin` = sesión de super_admin · `super_admin o provisioning` = además acepta la [credencial de provisioning](../authentication.md#3-credencial-de-provisioning) (`Authorization: Provisioning didprov_…`). Las marcadas **Cloud** solo existen cuando el operador enciende la función (el modo plataforma de cobros o la subida de vídeo); sin ella, la ruta responde 404.

!!! info "La lista blanca de la credencial"
    La credencial de provisioning **solo abre** las rutas marcadas «provisioning» de esta página (más `GET /admin/system/health-detail`). Cualquier otra ruta responde `403 AUTH_PROVISIONING_ROUTE_NOT_ALLOWED`, aunque la credencial sea válida. Renombrar una organización, ver la capacidad de la licencia o tocar la licencia exigen un super_admin con sesión.

## Organizaciones — `/admin/tenants`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/admin/tenants` | super_admin o provisioning | Lista todas las organizaciones (con nº de usuarios, cursos y dominios). |
| GET | `/admin/tenants/capacity` | super_admin | Nº de organizaciones y si la licencia permite crear más: sin `feat:multi_tenant.real` el límite es **1**. |
| GET | `/admin/tenants/usage?asOf=` | super_admin o provisioning | **Miembros activos por organización** a una fecha de corte (`asOf`, ISO-8601, por defecto ahora; una fecha futura da 400). Es la definición contractual del consumo —la misma de la página de precios de Didacta Cloud—. Cuenta: usuarios con acceso, activos y no borrados (formadores, auditores y gestores de empresa **sí** cuentan). No cuenta: `tenant_admin`/`super_admin`, invitaciones sin aceptar (`PENDING`), suspendidos, bajas y borrados. Devuelve `[{ tenantId, slug, activeMembers, asOf }]`. |
| GET | `/admin/tenants/:id` | super_admin o provisioning | Detalle de una organización. |
| POST | `/admin/tenants` | super_admin o provisioning | Crea organización + primer `tenant_admin` + dominio primario y envía el correo de bienvenida (el enlace apunta al dominio de **esa** aula). **402** al superar el límite Community sin `feat:multi_tenant.real`. Ver cuerpo abajo. |
| PATCH | `/admin/tenants/:id/status` | super_admin o provisioning | `{ status: ACTIVE \| SUSPENDED \| ARCHIVED }`. `SUSPENDED` y `ARCHIVED` invalidan las sesiones de todos sus usuarios. |
| PATCH | `/admin/tenants/:id` | super_admin | Renombra (`{ name }`); afecta a la firma de los correos («Equipo {name}»). No invalida sesiones. |
| PATCH | `/admin/tenants/:id/signups` | super_admin o provisioning | Congela o descongela las altas nuevas: `{ frozen, reason }` (`reason` de 5 a 280 caracteres; se le enseña al visitante que se queda fuera). Con las altas congeladas, la invitación individual, el alta masiva, el registro público y el checkout de alumno nuevo dan **409 `TENANT_SIGNUPS_FROZEN`**. Es un interruptor explícito: el núcleo no cuenta miembros ni lo enciende solo. |
| PUT | `/admin/tenants/:id/subscription-gate` | super_admin o provisioning | Abre, pone en prueba o cierra la [puerta de suscripción](#puerta-de-suscripcion). |
| POST | `/admin/tenants/:id/domains` | super_admin o provisioning | Añade un dominio (`{ hostname }`). |
| DELETE | `/admin/tenants/:id/domains/:hostname` | super_admin o provisioning | Quita un dominio; el primario no se puede quitar. |

**Cuerpo de `POST /admin/tenants`**: `slug` (≤64), `name` (≤120), `adminEmail`, `adminName?`, `primaryHostname`, y opcionales retrocompatibles `simpleMode` (sin él, nace **encendido** si crea el plano de control y **apagado** si crea un super_admin), `locale` (`es`, `en`, `pt` o un tag como `pt-BR`, que se guarda reducido; un idioma sin catálogo da 400; también fija el idioma de la primera administradora) y `payoutsCountry` (país del banco declarado en el alta, ISO alfa-2 u `OTHER`; solo se usa para preseleccionar en la tarjeta de cobros).

```bash
curl -X POST https://tu-instancia/api/v1/admin/tenants \
  -H "Authorization: Provisioning didprov_..." \
  -H "Content-Type: application/json" \
  -d '{
    "slug": "demo",
    "name": "Academia Demo",
    "adminEmail": "admin@example.com",
    "primaryHostname": "demo.example.com",
    "locale": "es"
  }'
```

## Puerta de suscripción

Una dimensión **aparte** del estado de la organización. `ACTIVE` dice que el aula existe y se sirve; la puerta dice si, además, su contenido está abierto, en prueba o en pausa. Un aula con la prueba terminada sigue `ACTIVE` a propósito: su administrador tiene que poder entrar, ver que está en pausa y pagar. El núcleo **no es una política**: no cuenta días ni mira planes; quien mueve la puerta es el plano de control o un super_admin. Una instalación autoalojada que nunca recibe esta llamada se queda en `OPEN`.

`PUT /admin/tenants/:id/subscription-gate` con `{ state, trialEndsAt?, actionUrl? }`:

| `state` | Qué pasa | Campos |
|---|---|---|
| `OPEN` | Todo como siempre. Borra `trialEndsAt` y `actionUrl`. | — |
| `TRIAL` | El aula funciona y sus administradores ven una franja con los días que quedan y el botón «Elegir plan» → `actionUrl`. | `trialEndsAt` (ISO-8601 con zona) y `actionUrl` (**https**) obligatorios. |
| `LOCKED` | Pantalla de pausa para todos y **402 `SUBSCRIPTION_LOCKED`** en la API, salvo acceso, `/me/profile`, `/me/modules`, marca, sondas, webhooks entrantes y rutas de organizaciones. | `actionUrl` (**https**) obligatorio. |

Un `TRIAL` con `trialEndsAt` ya pasado se comporta como `LOCKED` aunque la llamada no se repita. La llamada es idempotente: repetirla no cambia `updatedAt` ni deja otra fila en el audit log. `GET /auth/tenant-context` enseña al visitante solo si el aula está en pausa (`subscriptionGate.state: LOCKED`); la franja de prueba la ven solo los administradores, en `/me/modules`.

## Acceso de soporte

Una ventana temporal para que el equipo de soporte entre en el aula de un cliente sin hacerse pasar por nadie.

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/admin/tenants/:id/support-access` | super_admin o provisioning | Abre la ventana. Cuerpo `{ reason, ttlSeconds? }`: `reason` obligatorio (10–280 caracteres) —es lo que el cliente lee en **su** audit log—; `ttlSeconds` puede pedir **menos** de 15 minutos, nunca más (mínimo 60 s). Devuelve un token de **un solo uso** y el enlace para canjearlo. |
| DELETE | `/admin/tenants/:id/support-access/:grantId` | super_admin o provisioning | Corta la ventana antes de que caduque y cierra su sesión. Idempotente: la segunda vez devuelve `revoked: false`. |

- El canje (`POST /auth/support-access/redeem`, ver [Núcleo](core.md#acceso-de-soporte-authsupport-access)) abre sesión como un **usuario de soporte propio del aula**, visible en su lista de miembros; nunca como una persona real. La sesión no tiene refresh token y caduca con la concesión.
- Quedan dos filas en el audit log **del aula**: la concesión (con el motivo y quién la abrió) y el canje. El cliente las lee sin pedir permiso.
- Mientras la sesión vive, el aula muestra un aviso permanente que no se puede cerrar. Una sesión de soporte no puede exportar datos personales (`/admin/sales/export` da 403 `SALES_SUPPORT_SESSION_FORBIDDEN`).

## Credencial de provisioning — `/admin/provisioning-credential` (super_admin)

| Método | Ruta | Qué hace |
|---|---|---|
| POST | `/admin/provisioning-credential` | Emite la credencial de la instancia. Devuelve `{ id, token, preview, createdAt }`; el token `didprov_…` **solo se ve esta vez**. Emitir otra **revoca la anterior** (hay como mucho una viva). |
| DELETE | `/admin/provisioning-credential` | Revoca la credencial viva (idempotente): `{ revoked, id }`. |

Cómo se usa y qué cubre: [Autenticación](../authentication.md#3-credencial-de-provisioning).

## Cobros de plataforma — rutas del pool (Cloud)

Solo existen con el **modo plataforma** encendido (`PAYMENTS_PLATFORM_STRIPE_SECRET_KEY` en el entorno de la API); en cualquier otra instalación responden 404. Las usa el plano de control para saber cuánto se cobra y fijar la comisión. Lo que ve cada creador está en [Pagos, aula e IA](payments-live-ai.md#cobros-de-plataforma-modulespayouts-cloud).

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/admin/tenants/payouts/volume?from=&to=` | super_admin o provisioning | Volumen cobrado por la plataforma, **por aula, día (UTC del cobro) y divisa de liquidación**, entre `from` y `to` (ambos incluidos, `YYYY-MM-DD`, máx. 366 días). Una venta cuenta cuando su reparto se conoce: conviene volver a pedir los últimos días. Nunca suma divisas distintas. |
| GET | `/admin/tenants/payouts/countries` | super_admin o provisioning | Países (ISO alfa-2) donde la plataforma puede pagar a creadores: `PAYMENTS_PLATFORM_SUPPORTED_COUNTRIES` o la lista por defecto. Una sola fuente para Cloud y para el alta del creador. |
| PUT | `/admin/tenants/:id/payouts-settings` | super_admin o provisioning | Fija la **comisión del operador** de un aula: `{ feeBps }` en puntos básicos, de 0 a 3000 (500 = 5 %). Idempotente; queda en el audit log. |
| GET | `/admin/tenants/:id/payouts` | super_admin o provisioning | Modo de cobro del aula, estado de la cuenta conectada, comisión y deuda abierta. |

## Vídeo — rutas del pool (Cloud)

Solo existen con la subida de vídeo configurada (`BUNNY_STREAM_API_KEY`). El lado del creador está en [Aprendizaje → Vídeo](learning.md#video).

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/admin/tenants/:id/video` | super_admin o provisioning | Cuota de horas, última actividad, avisos de inactividad enviados, purga y copias de seguridad (nº y bytes) del vídeo subido de un aula. |
| POST | `/admin/tenants/:id/video-backups/purge` | super_admin o provisioning | **Supresión (RGPD)**: borra ya, de forma definitiva, todas las copias en Google Drive de los vídeos del aula. Idempotente. Devuelve `{ deleted, failed }`. |

## Licencia — `/admin/license` (super_admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/license` | Estado actual de la licencia de la instalación. |
| PUT | `/admin/license` | Activa una licencia: `{ key }`. Valida la firma antes de guardar y rechaza si no es válida. |
| DELETE | `/admin/license` | Quita la licencia guardada en el panel (vuelve a Community, o a la del entorno si hay). |
| POST | `/admin/license/refresh` | Re-evalúa el estado (expiración, gracia) sin cambiar la clave. |

El estado público —sin secretos— está en `GET /api/license`. Qué desbloquea cada capability: [Enterprise → Capabilities](../../enterprise/capabilities.md).

## Ajustes de instancia — `/admin/instance-settings` (super_admin)

Parejas `scope`/`key` globales de la instalación (no de un aula). Los secretos se cifran en reposo y **nunca devuelven su valor**, solo `hasValue`.

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/admin/instance-settings` · `/:scope` | Lista todos los ajustes de instancia (metadatos, sin valores secretos) · los de un scope. |
| GET | `/admin/instance-settings/:scope/:key` | Lee un ajuste. |
| PUT | `/admin/instance-settings/:scope/:key` | Crea o actualiza (`{ value, isSecret }`). |
| DELETE | `/admin/instance-settings/:scope/:key` | Elimina el ajuste. |

## Estado y registro de la instalación

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/admin/system/health-detail` | super_admin, tenant_admin o provisioning | Estado consolidado de BD, Redis, S3, SMTP, las dos colas y la versión desplegada (diagnóstico on-call). |
| GET | `/admin/registry/status` | super_admin | Estado del registro opt-in de la instalación con el equipo de Didacta: si está registrada, fecha de opt-in, último envío de telemetría y si hay conexión. |
| POST | `/admin/registry/opt-in` | super_admin | Registra la instalación y activa el envío de telemetría agregada (sin datos personales). `acceptTerms: true` obligatorio. |
| DELETE | `/admin/registry/opt-in` | super_admin | Opt-out (RGPD): marca el registro como cancelado y solicita el borrado remoto (best-effort si el servicio no responde). |

El registro es una decisión **de instancia**, no de aula. Ver [Telemetría](../../configuration/telemetry.md).

## Módulos de marketplace

La instalación, desinstalación e inspección de paquetes ZIP firmados (`/admin/modules/install`, `/admin/modules/installed…`) son operaciones de `super_admin` sobre la instancia y están en [Administración → Módulos](administration.md#modulos-adminmodules). Una vez instalado, un módulo de terceros atiende **todo lo que cuelga de `/modules/<slug>/…`** a través de un despachador genérico (`ALL /modules/*`): la autenticación es opcional en el despachador y cada ruta del módulo decide si exige sesión. El bundle de interfaz de cada módulo se sirve sin autenticación en `GET /modules/:slug/ui/:surface.js`.
