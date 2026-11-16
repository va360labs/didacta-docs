# Cobros y pagos

En Didacta Cloud **tus alumnos pagan en Didacta y Didacta te transfiere tu parte**. Tú no pegas claves de Stripe ni configuras webhooks: conectas una cuenta de cobros en unos 5 minutos y el resto lo hace la plataforma.

## Cómo circula el dinero

1. **El alumno paga con tarjeta** dentro de tu página de venta, con tu logo (o tus iniciales) y tu color. Stripe procesa el pago y le llega el recibo por correo. Puede pagar desde cualquier país.
2. **Figuras tú como vendedor**: en la página de pago, en el recibo y en el extracto bancario del alumno. Didacta cobra en tu nombre con su cuenta de Stripe y se queda su comisión (el [5 %](pricing.md)) y la de la tarjeta.
3. **Didacta transfiere el resto a tu cuenta de cobros de Stripe**, normalmente en minutos y como mucho en unas horas desde que Stripe confirma el cobro.
4. **Stripe envía el saldo a tu banco una vez por semana, los lunes**, con el dinero de las ventas que tienen al menos unos 7 días, y dejando siempre en tu cuenta de cobros un saldo mínimo de **50 €** (o su equivalente en tu moneda) para cubrir reembolsos y disputas. En la práctica, el dinero suele llegar a tu banco entre **7 y 14 días** después de la venta.

!!! info "Didacta no es una entidad de pago"
    El dinero se mueve en todo momento dentro de Stripe, que es una entidad autorizada, y tu parte no pasa por las cuentas bancarias de Didacta. Didacta actúa solo por tu cuenta, nunca por cuenta del alumno: cuando el alumno paga, ha pagado.

## Conectar tus cobros

Es el cuarto paso de [«Empieza aquí»](getting-started.md#3-empieza-aqui); también puedes hacerlo desde **Ventas**.

1. Pulsa **Conectar cobros**.
2. Elige tu **país** (el de tu cuenta bancaria) y **cómo vas a cobrar**: particular o autónomo, o empresa.
3. Se abre el alta de Stripe, que te pide tus datos, un documento de identidad y tu cuenta bancaria, y que aceptes sus condiciones. Los cobros para creadores los presta Stripe y están sujetos al [Acuerdo de cuentas conectadas de Stripe](https://stripe.com/connect-account/legal/full).
4. Vuelves al aula y ves en qué punto estás:

| Mensaje del aula | Qué significa |
| --- | --- |
| *Falta un dato para poder pagarte: …* | Stripe necesita algo más (tu cuenta bancaria, un documento, tus datos personales o los de tu actividad, aceptar sus condiciones). Pulsa **Completar**. |
| *Stripe está revisando tus datos* | No tienes que hacer nada: cuando termine, podrás vender. |
| *Stripe no ha aprobado esta cuenta* | De momento no puedes vender. Escribe al [soporte](support.md) y se revisa. |
| *Todo listo. Cobras en … y te pagamos cada lunes* | Ya puedes vender. **Ver mis pagos en Stripe** abre tu panel de Stripe. |

Hasta que Stripe da el visto bueno, el aula no deja comprar tus productos. Stripe te pedirá un **teléfono de atención al cliente**: tus alumnos lo verán en sus recibos; si no quieres dar tu móvil personal, pon uno de trabajo.

## Países y monedas

### Dónde puedes cobrar

Puedes cobrar si **resides y tienes tu cuenta bancaria** en uno de estos 33 países: los 27 de la **Unión Europea** (España incluida), **Noruega**, **Liechtenstein**, el **Reino Unido**, **Suiza**, **Estados Unidos** y **Canadá**.

Hoy **no** se puede pagar a creadores con el banco en México, Brasil, Argentina, Colombia, Chile, Perú ni en el resto de Latinoamérica. Puedes abrir tu aula igualmente y montar tu comunidad y tus cursos gratuitos, pero no cobrar ventas. La lista depende de dónde permite Stripe que Didacta pague; si cambia, se actualiza.

Tus **alumnos** pueden pagar desde cualquier país.

### En qué moneda cobras

- Tu **moneda de cobro** es la de tu cuenta de Stripe, según tu país: euros, libras, dólares estadounidenses o canadienses, francos suizos, coronas (sueca, danesa, noruega, checa), złotys, forintos o leus.
- Puedes poner el precio en cualquiera de esas doce monedas. Si es la de tu cuenta, el 5 % sale exacto y Stripe puede enseñar a cada alumno el precio convertido a su moneda (esa conversión la paga el alumno, no tú).
- Si eliges **otra** moneda distinta de la de tu cuenta, Stripe cobra un coste de conversión que paga el creador y el alumno ya no ve el precio convertido. La pantalla de precio te lo avisa.
- Además puedes fijar un **precio propio para otros mercados** (hasta 20 monedas extra).

## Tus ventas

En **Ventas** ves:

- **El resumen por moneda:** vendido, ya en tu banco, en camino y retenido.
- **Cada venta** con quién compró, qué, cuánto pagó, la comisión de la tarjeta, el 5 % de Didacta, lo que te llega y cuándo llega. Puedes filtrar por fechas, producto, estado o texto.
- **Exportar** las ventas a CSV o Excel y los compradores a CSV (con los filtros que tengas puestos; son datos personales: guárdalos con cuidado).

Estados que verás en cada venta:

| Estado | Significa |
| --- | --- |
| Cobrada · calculando comisiones | El alumno ha pagado; se está calculando el reparto. |
| Enviándote el dinero / Enviada a tu cuenta | La transferencia a tu cuenta de cobros está en camino o hecha. |
| Retenida | El dinero está a salvo pero no se transfiere todavía: faltan tus [datos fiscales](refunds-and-taxes.md#datos-fiscales), o Stripe no ha terminado de verificar tu cuenta, o se reintenta al día siguiente. |
| Reembolsada / Reembolsada en parte | Ver [Reembolsos y disputas](refunds-and-taxes.md#reembolsos). |
| Disputada · Reclamación ganada / perdida | El alumno reclamó el pago a su banco. Ver [Disputas](refunds-and-taxes.md#disputas). |
| Se usó para saldar lo que debías | Esa venta se aplicó a una [deuda](refunds-and-taxes.md#lo-que-debas) pendiente. |

## Cuándo puede retrasar Didacta una transferencia

Solo en estos casos, y siempre te dice el motivo salvo que la ley lo prohíba:

- Mientras no hayas completado tus datos fiscales.
- Mientras Stripe no haya terminado de verificar tu cuenta o la tenga restringida.
- Si hay indicios razonables de fraude o de ventas no autorizadas.
- Si tus reembolsos o disputas son anormalmente altos.
- Si lo pide una autoridad o un juez.
- Si incumples gravemente las condiciones.

La retención dura lo imprescindible y como máximo **hasta 120 días** después de la última venta afectada, que es el plazo en el que un alumno puede abrir una disputa.

## Códigos de descuento

Los códigos de descuento de Stripe **no funcionan** en las aulas de Didacta Cloud por ahora: la página de pago los rechaza. Lo que sí hay es el [descuento por «Invita y gana»](referrals.md) en la cuota de la membresía.

## Qué no cambia por usar Cloud

El alumno compra igual que en Community: página de venta del curso (`/catalog/<curso>`) o de la membresía (`/join`), y el acceso se concede solo al pagar. Los [módulos de cobro](../modules/catalog/billing.md) son los mismos; la diferencia está en **quién cobra** (la plataforma, no una clave tuya).
