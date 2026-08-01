# mod.referrals — Programa de referidos

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Programa de referidos de la membresía: cada miembro tiene un código y enlace propios (`/unete?ref=CODIGO`) y, si alguien entra por él y **paga**, el referidor devenga un porcentaje del cobro real. El operador configura todo desde el panel sin tocar código: porcentaje (`commissionBps`), ámbito (solo primer pago o recurrente con límite de meses), ventana de atribución, periodo de garantía, mínimo de liquidación y si exige membresía activa del referidor.

## Cómo funciona

El ciclo completo: clic (deduplicado por código + día + **hash de la IP** — la IP nunca se guarda en claro) → checkout con el código en la metadata de Stripe → **atribución** en el fulfillment del webhook (idempotente y anti-auto-referido) → comisión `PENDING` por cada factura pagada dentro del ámbito → `APPROVED` al vencer la garantía (worker automático) → **liquidación manual** del admin con referencia externa → `PAID`.

- Un reembolso **revoca automáticamente** la comisión en `PENDING`/`APPROVED`; una ya `PAID` no se toca (clawback manual del operador).
- El referidor recibe notificación in-app + email al ganar comisión y al cobrar liquidación (plantillas editables en Administración → Emails).
- Con el programa desactivado, los endpoints públicos se comportan como si no existiera (404).

## Dependencias

Opcional: `mod.subscriptions` (comprobar membresía viva del referidor).

## Modelo de datos

`mod_referrals_config` (política por tenant) · `mod_referrals_code` (código único por usuario) · `mod_referrals_click` (clics deduplicados) · `mod_referrals_referral` (atribución) · `mod_referrals_commission` (única por factura de Stripe) · `mod_referrals_payout` (liquidaciones).

## API

Prefijo `/modules/referrals` (miembro + `track` público + admin). Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#referidos-modulesreferrals).

## Eventos

- **Emite**: `referrals.referral.attributed`, `referrals.commission.created/approved/revoked`, `referrals.payout.recorded`.
- **Consume** (vía bridge): `subscriptions.membership.activated`, `subscriptions.invoice.paid`, `subscriptions.invoice.refunded`.

## Configuración

Todo por tenant en Administración → Referidos. Sin variables de entorno propias (cron de aprobación: `REFERRALS_APPROVAL_CRON`). v1 sin Stripe Connect: la liquidación se registra a mano con referencia externa.
