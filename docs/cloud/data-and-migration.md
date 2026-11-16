# Baja y migración de datos

Tus contenidos y los datos de tus alumnos son tuyos. Esta página explica cómo te los llevas, qué pasa si te das de baja y cómo pasar de Didacta Cloud a una instalación propia (Community) y al revés.

## Llevarte tus datos

Hoy **no hay un botón para descargar tu aula entera**. Se pide por correo:

1. Escribe a [hola@didacta.io](mailto:hola@didacta.io) pidiendo una copia de tu aula.
2. En un plazo máximo de **30 días** y **sin coste**, Didacta te la entrega en formatos de uso común:
    - hojas de cálculo con tus **alumnos**, sus **inscripciones**, tus **ventas** y las **publicaciones de tu comunidad**;
    - los **textos y archivos de tus cursos**;
    - los **vídeos que subiste** a Didacta.

Lo que sí puedes sacar tú, cuando quieras, desde **Ventas**: las ventas en CSV o Excel y los compradores en CSV (con los filtros que tengas puestos). Son datos personales: guárdalos con cuidado.

## Darte de baja

- Puedes terminar el contrato **cuando quieras y sin coste**, escribiendo a [hola@didacta.io](mailto:hola@didacta.io).
- **Al terminar** dejas de poder vender. Las suscripciones vivas de tus alumnos se cancelan **al final del periodo que ya han pagado**, sin nuevos cobros.
- Tu **cuenta de cobros de Stripe sigue abierta 120 días** para poder atender reembolsos y disputas. Pasado ese plazo, y una vez saldadas las deudas, se te paga el saldo que quede, incluido el saldo mínimo.
- Tu aula **se conserva 30 días** para que puedas pedir tu copia. Después se borra, salvo lo que la ley obligue a guardar (por ejemplo, el registro de ventas y comisiones, 6 años).
- Si Didacta quiere terminar el contrato, te avisa **con al menos 30 días** y te dice por qué (salvo causas graves, que se explican en las [condiciones](https://didacta.io/es/legal/terms), apartado 13). Si dejara de ofrecer Didacta Cloud para todos, también te avisa con 30 días como mínimo.

!!! info "Aulas archivadas por inactividad"
    Si tu aula se archivó por [no tener actividad](video-and-inactivity.md#aulas-sin-actividad), tus alumnos y tu historial de ventas siguen ahí: escribe al mismo correo para reabrirla.

## De Cloud a Community (self-hosted)

Cloud ejecuta la **misma imagen publicada** que se instala en Community, así que todo lo que tienes en tu aula existe también en una instalación propia. Lo que no existe es un migrador: Cloud no tiene ninguna herramienta de migración por ahora, así que el traspaso es manual. El camino:

1. **Pide la copia** de tu aula (arriba).
2. **Instala Community** con [Docker Compose](../installation/docker-compose.md) y haz el [asistente de configuración](../installation/setup-wizard.md).
3. **Recrea tu contenido** con los textos y archivos de la copia: cursos, lecciones, comunidad.
4. **Trae a tus alumnos**: invítalos desde el panel, uno a uno o por lotes. Reciben un correo para crear su contraseña.
5. **Conecta tus pagos**: en Community cobras con tu propia cuenta de Stripe, sin la comisión del 5 % de Cloud. Mira [Conexiones de pago](../modules/catalog/payment-connections.md) y [Facturación](../modules/catalog/billing.md).

!!! warning "Las membresías de pago no se traspasan solas"
    En Cloud el cobro de tus alumnos se hace en la cuenta de Stripe de Didacta y las suscripciones se cancelan al final del periodo pagado cuando te das de baja. Los alumnos con membresía tendrán que volver a suscribirse en tu nueva instalación. Si quieres, dales acceso a mano en tu instalación durante la transición.

## De Community a Cloud

Cloud crea un **aula nueva** en [didacta.io/es/start](https://didacta.io/es/start). No tiene migrador por ahora: no existe un asistente que mueva una instalación Community entera ni que importe datos de otras plataformas, y las copias de seguridad de la base de datos sirven para restaurar *esa* instalación, no para importarlas en Cloud. El camino:

1. Abre tu aula de Cloud y recorre [Empezar](getting-started.md).
2. Recrea tus cursos y trae a tus alumnos por invitación o por CSV.
3. Para los alumnos que ya te pagaron en tu instalación, puedes darles **acceso a mano** sin que paguen de nuevo ni se genere comisión: es un uso permitido para alumnos que ya te compraron en otra plataforma. Lo que no se permite es usarlo como forma habitual de vender fuera de Didacta.
4. Conecta tus [cobros](payouts.md) para las ventas nuevas. Si quieres, puedes ir por tramos: empezar tu próximo curso o grupo en Cloud y dejar el resto donde está.

## Si Didacta cerrara

El código de Didacta es público: puedes instalarlo en tu propio servidor y seguir. Cloud y Community son el mismo producto, y esa es la razón de que el camino de [Cloud a Community](#de-cloud-a-community-self-hosted) exista.
