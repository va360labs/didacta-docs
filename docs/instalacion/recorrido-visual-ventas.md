# Recorrido visual: notificaciones y ventas

Segunda parte del recorrido visual — continúa donde termina [Recorrido visual: primeros pasos](recorrido-visual.md). Aquí configuramos el correo saliente y preparamos la instancia para vender: cursos sueltos y membresías.

!!! note "De dónde salen estas capturas"
    Misma instancia aislada y sin seeds de la primera parte. El SMTP apunta al Mailpit del propio `docker-compose.alpha.yml` — es un envío real, no una simulación: el correo de prueba llega de verdad al buzón.

## 1 · Notificaciones (SMTP)

En **Configuración → Notificaciones** (`/admin/configuracion`, tab "Notificaciones") cada tenant puede definir su propio servidor SMTP. Sin esto, la organización hereda el SMTP global de la instancia si el operador lo configuró por variables de entorno — ver [Email](../configuracion/email.md).

![Tab Notificaciones vacío, sin SMTP configurado](../assets/notificaciones-y-pagos/01-smtp-vacio.png)

Rellena host, puerto, usuario/contraseña y el remitente. Importante: la **"Conexión segura (TLS)" viene activada por defecto** — solo la necesitas si tu proveedor exige STARTTLS/SSL (típicamente puerto 587 o 465). Un servidor sin cifrado en el puerto 25/1025 (como Mailpit en desarrollo) necesita el TLS **apagado**, o la conexión falla con un error de handshake.

![Formulario SMTP relleno, TLS desactivado para un servidor sin cifrar](../assets/notificaciones-y-pagos/02-smtp-form.png)

Al guardar, el banner pasa a "configurado pero sin verificar" — Didacta no prueba la conexión al guardar, solo valida el formato.

![Banner "configurado pero sin verificar" tras guardar](../assets/notificaciones-y-pagos/03-smtp-guardado.png)

**"Enviar email de prueba"** confirma que la configuración funciona de verdad: abre un diálogo con el email del propio admin prellenado.

![Diálogo de envío de prueba con el destinatario prellenado](../assets/notificaciones-y-pagos/04-smtp-modal-prueba.png)

Si el correo llega, el banner pasa a **"Verificado"** con fecha y hora — la señal de que la organización ya puede mandar invitaciones, resets de contraseña y avisos de curso de verdad.

![Banner "Verificado" con fecha y hora](../assets/notificaciones-y-pagos/05-smtp-verificado.png)

Y el correo, real, en el buzón:

![El email de prueba, real, en la bandeja de Mailpit](../assets/notificaciones-y-pagos/06-mailpit-bandeja.png)

!!! tip "Editar el contenido de los emails"
    El formulario de arriba solo configura el **transporte**. El texto de cada email (asunto, cuerpo) se edita por tenant en **Administración → Emails**, sin tocar código — ver [Email](../configuracion/email.md).

## 2 · Vender cursos sueltos (mod.billing)

Stripe se configura **por tenant, desde el panel** — no hace falta tocar el `.env` de la instancia. Un único par de credenciales (clave secreta + secreto de webhook) sirve tanto para cursos sueltos (`mod.billing`) como para suscripciones/membresía (`mod.subscriptions`): comparten cuenta de Stripe, la misma academia cobra por los dos caminos.

En **Configuración → Pagos** (`/admin/configuracion`, tab "Pagos"), sin nada configurado todavía:

![Tab Pagos vacío, sin Stripe configurado](../assets/notificaciones-y-pagos/07-pagos-vacio.png)

Mientras Stripe no esté configurado, dos pantallas más lo avisan **antes** de que un alumno intente pagar, en vez de dejar que el fallo aparezca en producción sin más contexto:

- **Membresía** (`/admin/membresia`) muestra un aviso arriba del todo — planes y la página `/unete` se preparan igual, pero el cobro real fallará hasta que configures Stripe.

  ![Aviso de Stripe sin configurar en /admin/membresia](../assets/notificaciones-y-pagos/08-membresia-aviso-stripe.png)

- **Pagos → Productos Stripe** (`/admin/billing/products`) deja vincular cursos igual, pero al guardar sin Stripe configurado responde con un mensaje claro y un enlace directo a la pestaña Pagos — nunca un 500 crudo.

  ![Error con enlace a Administración → Pagos al vincular un curso sin Stripe configurado](../assets/notificaciones-y-pagos/09-billing-cta-sin-stripe.png)

Para activarlo: pega la **clave secreta** (Stripe → Desarrolladores → Claves de API) y el **secreto del webhook** — la propia tarjeta enseña las dos URLs a pegar en tu panel de Stripe (`{tu-dominio}/api/v1/modules/billing/webhook` y `.../modules/subscriptions/webhook`; el webhook de suscripciones es opcional, si lo dejas vacío reutiliza el mismo secreto que cursos sueltos).

![Formulario de Pagos con credenciales de prueba, antes de guardar](../assets/notificaciones-y-pagos/10-pagos-form-relleno.png)

Al guardar, igual que SMTP, el banner pasa a "configurado pero sin verificar" — Didacta no llama a Stripe al guardar, solo valida el formato.

![Banner "configurado pero sin verificar" tras guardar](../assets/notificaciones-y-pagos/11-pagos-guardado-sin-verificar.png)

**"Probar conexión"** sí llama a la API real de Stripe (`balance.retrieve`, de solo lectura, sin efectos secundarios). Con una clave de verdad marca el banner como verificado; aquí, con una clave inventada para esta demo, Stripe la rechaza con su propio mensaje de error — la señal de que la validación es real, no cosmética:

![Stripe rechaza una clave inventada con su propio mensaje de error](../assets/notificaciones-y-pagos/12-pagos-probar-conexion-error.png)

Con una clave real guardada y verificada: en cada curso vinculado desde **Pagos → Productos Stripe**, eliges el curso publicado, pegas el `price_id` de un **Product + Price ya creado en tu dashboard de Stripe** (Didacta valida contra la API que existe y está activo antes de guardar) y el botón «Comprar curso» aparece solo en el catálogo público de ese curso.

!!! warning "No confundir con «Conexiones de pago»"
    `/admin/integraciones/payment-connections` es una pantalla **distinta**: sirve para reconciliar suscriptores de una cuenta de Stripe **externa** con una clave restringida de solo lectura — no habilita el checkout de Didacta. Para vender de verdad, la única vía es configurar Stripe en **Administración → Pagos**, de arriba.

!!! tip "Instalaciones que ya usaban STRIPE_SECRET_KEY por .env"
    Sigue funcionando como fallback de instancia: si un tenant no configura sus propias credenciales en el panel, hereda las variables de entorno del operador (si están definidas) — igual que el SMTP global. Ningún despliegue existente se rompe al actualizar.

## 3 · Vender membresías (mod.subscriptions)

A diferencia de los cursos sueltos, **crear y editar planes de membresía no necesita Stripe configurado** — el Price de Stripe se genera solo en el primer cobro real. Solo el checkout final (el botón "Continuar al pago seguro") depende de las mismas variables del paso anterior.

### Grupo de acceso

Cada plan concede acceso a través de un **grupo de acceso**. Antes de crear el plan, damos de alta uno en `/admin/grupos-acceso` con tipo "Todos los cursos":

![Alta del grupo de acceso "Acceso completo" (todos los cursos)](../assets/notificaciones-y-pagos/13-grupo-acceso.png)

### El plan

En **Membresía** (`/admin/membresia`) creamos el plan: nombre, periodicidad, moneda, precio (y precio tachado opcional), días de prueba y si aparece preseleccionado en la página pública.

![Alta de un plan mensual: 19€ (antes 29€), 7 días de prueba](../assets/notificaciones-y-pagos/14-membresia-plan-form.png)

![Plan ya creado y listado](../assets/notificaciones-y-pagos/15-membresia-plan-creado.png)

### La página pública `/unete`

En la misma pantalla, la tarjeta "Página pública" controla el contenido de `/unete`: activarla, título, el grupo de acceso que otorga, subtítulo, y si muestra el catálogo de cursos incluidos.

![Configuración de la página pública /unete](../assets/notificaciones-y-pagos/16-unete-config.png)

![Página /unete activa tras guardar](../assets/notificaciones-y-pagos/17-unete-config-guardada.png)

Y el resultado, en vivo, sin sesión — con el precio, el ahorro, el curso incluido y las preguntas frecuentes ya generadas a partir de datos reales del tenant:

![/unete en vivo con el plan mensual visible](../assets/notificaciones-y-pagos/18-unete-publico.png)

El botón "Continuar al pago seguro" abre Stripe Checkout — funciona en cuanto configures Stripe en **Administración → Pagos** (§2), por tenant. Sin credenciales (ni del tenant ni del fallback de instancia), responde el mismo error explicado arriba.

## Siguiente paso

- [Email](../configuracion/email.md) — SMTP global vs. por tenant, plantillas.
- [mod.billing — Stripe Checkout](../modulos/catalogo/billing.md)
- [mod.subscriptions — Suscripciones](../modulos/catalogo/subscriptions.md)
- [mod.payment-connections — Conexiones de pago](../modulos/catalogo/payment-connections.md) — para no confundirlo con lo anterior.
