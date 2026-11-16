# mod.subscriptions — Suscripciones recurrentes

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Suscripciones recurrentes con Stripe (`mode=subscription`), con periodicidad configurable de 1 a 12 meses (mensual, trimestral, semestral, anual…): suscripción a cursos concretos y, sobre esa base, la **membresía** de la página pública `/join`, con planes propios (periodicidad, precio, precio tachado, días de trial, plan destacado) y su configuración de landing (titulares, grupo de acceso que concede, límite de lecciones en trial, testimonial).

## Cómo funciona

- Los webhooks de Stripe (`customer.subscription.*`, `invoice.*`) actualizan el estado local: `PENDING → TRIALING / ACTIVE → PAST_DUE → UNPAID / CANCELED`.
- **Periodo de gracia** configurable tras un impago (3 días por defecto): si Stripe cobra antes de expirar, vuelve a `ACTIVE`; si expira, pasa a `UNPAID` y **pausa la matrícula** (un cron horario expira las gracias vencidas). El dunning lo hacen los reintentos automáticos de Stripe. Un trial que nunca pagó no tiene gracia: si su primer cobro falla, pierde el acceso al momento.
- La cancelación puede ser al final del periodo (default) o inmediata (desenrola al momento) por API; el alumno, desde su cuenta, solo tiene **«Darme de baja»**, que abre la **oferta al darse de baja** (ver más abajo) y acaba en la baja al final del periodo pagado.
- Bridges del host conectan con el resto: activación → matrícula; impago → pausa; membresía activada/revocada → concesión/retirada del grupo de acceso.
- El checkout de membresía es **anónimo** (el visitante paga en `/join` y la cuenta se materializa con el email confirmado por Stripe) y acepta códigos de referido, que interpreta `mod.referrals`. No se vende una segunda membresía a quien ya tiene una viva (409 `MEMBERSHIP_ALREADY_SUBSCRIBED`).
- Como en la venta de cursos, sin la **casilla de renuncia al desistimiento** no se paga (422), y si la cuenta que cobra tiene **clave publicable** el pago se hace **dentro de `/join`** (Checkout embebido; vuelta `/join?status=success&session_id=…`, que pregunta a la API si se pagó y no concede nada por sí misma). Detalle en [mod.billing](billing.md).
- Con **cobros de plataforma** ([mod.payouts](payouts.md)), si la cuenta del creador aún no puede vender, `/join` se ve sin planes y el checkout responde 409 `PAYOUTS_NOT_READY`.
- Cada cobro genera un correo-recibo (compra, renovación, pago fallido) que el creador puede personalizar; ver [mod.billing → Correos de venta](billing.md#correos-de-venta). La suscripción guarda el país del comprador y el cupón aplicado, y aparece en la pantalla de **Ventas** ([mod.billing → Ventas](billing.md#ventas)).
- Convive con `mod.billing` en el mismo tenant (cursos one-shot y por suscripción) sin compartir tablas.

!!! note "Capturas en modo completo"
    Las capturas de esta página son del **modo completo** (menú lateral), el de una instalación nueva. En **modo sencillo** las mismas rutas siguen disponibles desde «Más», salvo las pantallas que ese modo sustituye por una versión simplificada (el editor de cursos, Ajustes, Ventas, Alumnos, Directos, la comunidad y Referidos); para usarlas tal como se ven aquí, cambia al modo completo en **Ajustes → «Usar el modo completo»**.

## Configuración

El módulo es de categoría **core**: viene activo en todos los tenants y no tiene interruptor propio. Nada de lo que sigue exige licencia Enterprise.

### Credenciales de Stripe (por tenant)

Las mismas de `mod.billing`: en `/admin/settings?tab=payments` (pestaña **Pagos**), tarjeta **«Pagos · Stripe»** (incluida la **«Clave publicable (opcional)»** para pagar dentro de `/join`). Este módulo añade un campo propio: **«Secreto del webhook de suscripciones (opcional)»** — si lo dejas vacío se usa el mismo secreto que el webhook de cursos sueltos. Sin credenciales (ni de tenant ni de instancia) el checkout responde 503 — planes, trial y grupo de acceso siguen editables igualmente, no dependen de Stripe.

### Webhook a dar de alta en Stripe

La tarjeta muestra la URL exacta. Para este módulo (recomendado como endpoint separado del de billing):

- Endpoint: `https://<dominio-de-tu-academia>/api/v1/modules/subscriptions/webhook`
- Eventos a seleccionar: `checkout.session.completed`, `checkout.session.expired`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`, `charge.refunded`.

!!! danger "`invoice.paid` e `invoice.payment_failed` sostienen todo el ciclo de vida"
    No son opcionales ni «para el historial». Si faltan:

    - **No hay corte por impago.** `invoice.payment_failed` es lo que marca la suscripción como vencida, arranca el periodo de gracia y acaba retirando el acceso. Sin él, quien deja de pagar **conserva la membresía indefinidamente** y en el panel figura como activa.
    - **El periodo de prueba no convierte.** Quien paga al acabar el trial se queda con el límite de clases de la prueba.
    - **El alumno no ve ninguna factura** en su cuenta, por muchos recibos que haya cobrado Stripe.

    Nada de esto da error en ningún log: el evento que no llega no se echa de menos. Si dudas, compruébalo al revés — cobra una vez en modo de prueba y mira que aparezca la factura en la cuenta del alumno.

### Formas de pago

Las que tengas activadas en tu panel de Stripe, igual que en la venta de cursos sueltos. Stripe filtra automáticamente las que no sirven para cobros recurrentes, así que la lista que ve el comprador de membresía puede ser más corta que la de un curso suelto.

### Planes y página de membresía

Todo en `/admin/membership` (**«Membresía»**). Si Stripe no está configurado, la página lo avisa («puedes preparar planes y página, pero el checkout real fallará…»).

- Tarjeta **«Planes»**: campos **«Nombre»**, **«Periodicidad»** (Mensual / Trimestral / Semestral / Anual / cada N meses), **«Moneda»**, **«Precio»**, **«Precio tachado (opcional)»**, **«Días de prueba»**, interruptor **«Preseleccionado en la página»**; botón **«Crear plan»**. El Product/Price de Stripe se crea perezosamente en el primer checkout y se **rota** al cambiar importe, moneda o periodicidad (los prices de Stripe son inmutables); renombrar el plan renombra el Product. Un plan con ventas no se borra: se desactiva. Por API un plan admite además precios fijos por mercado (`currencyOptions`, moneda → importe).
- Tarjeta **«Página pública»**: interruptor **«Página de compra activa»**, **«Título»**, **«Subtítulo»**, **«Grupo de acceso que concede»**, **«Mostrar el catálogo de cursos en la página»**, **«Clases visibles por curso durante el periodo de prueba»** (0 = sin límite), precios individuales de referencia por curso y testimonial opcional; botón **«Guardar configuración»**.

- Tarjeta **«Cuando alguien quiere irse»** (oferta al darse de baja; en modo sencillo está en Ajustes): ver la sección siguiente. Más abajo, **«Quién se quiere ir y por qué»** resume los últimos días.

![Administración de la membresía: planes y configuración de la página pública](../../assets/modules/subscriptions/subscriptions-1-membership-plans.png)

!!! note "La captura es anterior a la tarjeta de bajas"
    La captura muestra los planes y la página pública; no incluye las tarjetas **«Cuando alguien quiere irse»** y **«Quién se quiere ir y por qué»**, que se añadieron después.

### Oferta al darse de baja

Cuando un alumno pulsa **«Darme de baja»** en `/account?tab=subscription` (es el único camino de baja: el portal de Stripe ya no la ofrece), la ventana le pide el **motivo** (*Me sale caro*, *No tengo tiempo*, *No era lo que esperaba*, *Ya aprendí lo que quería*, *Faltan temas que me interesan*, *Otro motivo*, con texto libre opcional) y, si el aula lo ha activado, le ofrece quedarse:

- **Ofrecer una pausa**: sin pagar ni tener acceso al contenido; su cuenta, progreso y certificados se quedan y vuelve sola al terminar. Ajuste **«Como mucho»** de 1 a 3 meses (3 por defecto). Salta cobros enteros, así que en un plan mensual vale cualquier número de meses hasta el máximo, en uno trimestral solo 3 y en uno anual no se ofrece. Una pausa por persona cada 365 días.
- **Ofrecer un descuento**: **«Descuento»** del 5 al 90 % (30 por defecto) **«Durante»** 1 a 12 meses (3 por defecto). En planes mensuales se aplica a las siguientes cuotas; en trimestrales o anuales, a la siguiente renovación. Una vez por persona.

Ambas nacen **apagadas**. La tarjeta se guarda con **«Guardar la oferta»**. Lo que elige el alumno (pausa, descuento o baja al final del periodo) se confirma por correo, se ve en su pestaña (*Pausa programada…*, *En pausa: vuelve sola el…*, *…% de descuento hasta…*) y entra en el resumen del creador. Un barrido (el mismo worker horario de las gracias) empieza las pausas cuando llega su fecha y emite `subscriptions.subscription.paused`. API: `GET|PUT /membership/admin/retention`, `GET /membership/admin/retention/summary`, y para el alumno `GET|POST /modules/subscriptions/me/:id/leave`.

### Variables de entorno

| Variable | Para qué |
| --- | --- |
| `STRIPE_SECRET_KEY` / `STRIPE_PUBLISHABLE_KEY` | Fallback de instancia: solo si el tenant no configuró sus claves en el panel. Ver también `STRIPE_ALLOW_GLOBAL_FALLBACK` en [mod.billing](billing.md). |
| `SUBSCRIPTIONS_WEBHOOK_SECRET` | Webhook dedicado (recomendado separarlo del de billing); fallback de instancia. |
| `SUBSCRIPTIONS_GRACE_PERIOD_DAYS` | Gracia tras impago (3). |
| `SUBSCRIPTIONS_GRACE_EXPIRATION_CRON` | Cron que expira las gracias vencidas y arranca las pausas que llegan a su fecha (`0 * * * *`, cada hora UTC). |
| `SUBSCRIPTIONS_SUCCESS_URL_BASE` / `SUBSCRIPTIONS_CANCEL_URL_BASE` | Retorno del checkout de suscripción por curso (default: `/account/subscriptions` del dominio de la instancia, que redirige a `/account?tab=subscription`). |

El resumen diario y los avisos de renovación (`SUBSCRIPTIONS_DAILY_CRON`, `SUBSCRIPTIONS_DAILY_TZ`, `SUBSCRIPTIONS_RENEWAL_WINDOW_DAYS`) pertenecen a [mod.payment-connections](payment-connections.md), que es quien vigila las suscripciones externas.

Fuera del MVP: cambio de plan con prorrateo, facturación B2B y pause/resume — la cancelación es la única salida.

## Uso paso a paso

1. Configura Stripe en `/admin/settings?tab=payments` y da de alta el webhook de suscripciones con la URL y los eventos de arriba.
2. En `/admin/membership`, crea los planes: **«Nombre»**, **«Periodicidad»**, **«Precio»**, **«Días de prueba»** si quieres trial, y marca uno como **«Preseleccionado en la página»** → **«Crear plan»**.
3. En la tarjeta **«Página pública»**, activa **«Página de compra activa»**, elige el **«Grupo de acceso que concede»** (es lo que desbloquea los cursos al pagar) y, si usas trial, fija **«Clases visibles por curso durante el periodo de prueba»** → **«Guardar configuración»**.
4. Comparte `/join`. El visitante elige plan y pulsa **«Continuar al pago seguro»** («Pago procesado por Stripe · Cancela cuando quieras»; admite cupones en la pantalla de pago). Al volver (`/join?status=success`) ve «¡Pago recibido!» y recibe el email para **definir su contraseña**; la cuenta se crea sola y el grupo de acceso se concede vía evento.

5. El alumno gestiona su suscripción en `/account?tab=subscription` (pestaña **«Suscripción»**): fechas de prueba y próximo cobro, **«Pagar ahora y activar»** (termina el trial y cobra ya), **«Darme de baja»** (oferta al darse de baja, ver arriba) y **«Ver facturas»** (con enlace al PDF de Stripe; si pagó con saldo de referidos, la factura lo indica: «Pago con saldo de referidos»). Con cobros de plataforma aparece además **«Gestionar mi suscripción»** (portal de Stripe para cambiar la tarjeta y ver facturas).

6. Impagos: no tienes que hacer nada. Stripe reintenta el cobro; durante la gracia el alumno conserva el acceso (`PAST_DUE`) y, si la gracia vence sin pago, la matrícula se pausa (`UNPAID`) hasta que un cobro posterior la reactive.
7. La suscripción **por curso** (además de la membresía) existe hoy solo por API: `POST /modules/subscriptions/checkout/<courseId>` con un price recurrente de Stripe — la web no tiene botón para ella.

## Permisos

`subscriptions.subscription.read`, `subscriptions.subscription.create` y `subscriptions.subscription.cancel`.

## Dependencias

Duras: `mod.learning`, `mod.courses`.

## Modelo de datos

`mod_subscriptions_subscription` (una viva por usuario y curso) · `mod_subscriptions_plan` (planes de membresía + IDs de Stripe) · `mod_subscriptions_membership_config` (copy y reglas de `/join`) · `mod_subscriptions_invoice` · `mod_subscriptions_webhook_event` (log idempotente) · `mod_subscriptions_retention_config` (ajustes de la oferta al darse de baja) · `mod_subscriptions_exit` (cada baja: motivo y lo que eligió el alumno).

## API

Prefijos `/modules/subscriptions` (alumno + webhook) y `/membership` (público + admin). Detalle en [Referencia → Pagos](../../api/reference/payments-live-ai.md#suscripciones-modulessubscriptions) y [Referencia → Comunidad](../../api/reference/community.md#membresia-membership).

## Eventos

**Emite**: `subscriptions.subscription.created/activated/past_due/unpaid/canceled/paused`, `subscriptions.invoice.paid/payment_failed/refunded/partially_refunded`, `subscriptions.membership.activated`, `subscriptions.retention.paused/discounted/canceled`. No consume. No hay un `membership.revoked`: la pérdida de acceso de una membresía viaja en `subscriptions.subscription.canceled` (cancelación inmediata), `.unpaid` y `.paused` con `planId`, que es lo que escucha el puente de grupos.
