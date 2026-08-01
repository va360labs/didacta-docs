# mod.payment-connections — Conexiones de pago

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Conecta varias cuentas de pago **en modo solo lectura** (Stripe con clave restringida, y lectores de PayPal y WooCommerce) y reconcilia sus suscripciones activas contra los usuarios de Didacta **por email**. Sobre esa base construye:

- El catálogo de **tiers** (nivel manual asignado por el admin y/o derivado del pago), que otros módulos consumen (drip, grupos de acceso).
- Un **dashboard de control de suscripciones** materializado por un worker de sincronización, con enlaces de renovación y emails de recordatorio.
- El **espejo de pedidos** de tiendas externas (WooCommerce, con webhook firmado) y reglas de clasificación de accesos (`LIFETIME`, `SUBSCRIPTION`, `TIMED`…), con avisos de caducidad.
- La **invitación en bloque** de suscriptores que aún no están en Didacta.

## Qué NO es

No crea cobros: para vender están [mod.billing](billing.md) y [mod.subscriptions](subscriptions.md). Cancelar es un deep-link al portal de Stripe — el modelo es 100% read-only sobre las cuentas conectadas.

## Cómo funciona

- El match es por **email normalizado** (lowercase + trim) en ambos lados; cuentan como activas `active`, `trialing` y `past_due`.
- El tier efectivo se resuelve: manual → derivado del pago → «Desconocido». Los cambios publican `payment_connections.user_tier.changed`, que `mod.access-groups` consume para reconciliar membresías.
- Las claves API se guardan **cifradas** por conexión; jamás salen en un GET.
- El dashboard lee solo la tabla materializada — nunca consulta a los proveedores en vivo.

## Dependencias

Ninguna.

## Modelo de datos

`mod_payment_connections_connection` · `_log` · `_tier` · `_user_tier` · `_subscriber` (snapshot materializado) · `_order` (espejo de pedidos con entitlement y caducidad) · `_sync_history`.

## API

Prefijo `/modules/payment-connections` (super_admin salvo el autoservicio `me/*`). Detalle en [Referencia → Pagos](../../api/referencia/pagos-directo-ia.md#conexiones-de-pago-modulespayment-connections-super_admin).

## Eventos

**Emite**: `payment_connections.user_tier.changed`. No consume.

## Configuración

Sin variables propias: credenciales, secreto del webhook Woo, plantilla de renovación y URL del portal de cancelación son **por tenant**, cifradas en base de datos. Scopes mínimos de la clave restringida de Stripe: `Customers = Read` + `Subscriptions = Read` (opcional `Invoices = Read`). Cron de sincronización: `PAYMENT_SUBSCRIBERS_SYNC_CRON`.
