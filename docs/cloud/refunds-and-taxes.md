# Reembolsos, disputas e impuestos

Qué pasa cuando el dinero vuelve al alumno, qué datos fiscales necesitas para cobrar y qué te toca a ti como vendedor. Para el flujo normal del dinero, mira [Cobros y pagos](payouts.md).

## Reembolsos

Tú decides si devuelves dinero. Lo haces desde tu panel y Didacta te devuelve su parte proporcional; lo que Stripe no devuelve corre de tu cuenta.

### Cómo se reembolsa

1. En **Ventas**, abre la venta y pulsa **Reembolsar**.
2. Elige **todo el importe** o **una parte** y escribe cuánto.
3. Antes de confirmar, el aula te enseña qué recibe el alumno y cuánto se te descuenta a ti.

Nadie de Didacta reembolsa una venta tuya por su cuenta, salvo que lo exija la ley, una autoridad o Stripe, o para frenar un fraude evidente; en ese caso te avisa y se aplican las mismas reglas.

### Qué pasa con el dinero

- El dinero vuelve al alumno **por el mismo medio** con el que pagó; suele verlo en 5 a 10 días.
- Si ya te habían transferido tu parte, se **retira de tu saldo de Stripe** la parte proporcional. Si no hay saldo suficiente, la diferencia queda como [deuda](#lo-que-debas). Si aún no te la habían enviado, se resta de lo que ibas a recibir.
- Didacta **te devuelve su 5 %** en la misma proporción en que reembolsas.
- **Stripe no devuelve su comisión** cuando hay un reembolso. Esa comisión la asumes tú y se descuenta de tus próximos cobros.
- Con un reembolso **total**, el alumno pierde el acceso a lo que compró automáticamente. Con uno **parcial**, lo conserva, salvo que se lo quites tú.

**Ejemplo.** Reembolsas entera una venta de 100 € pagada con tarjeta española. El alumno recupera 100 €. Se revierten los 93,25 € que te transfirieron, Didacta te devuelve sus 5,00 € y la comisión de Stripe de 1,75 €, que no se recupera, se descuenta de tus próximos cobros. Si reembolsas 40 €, se revierten 37,30 € y Didacta te devuelve 2,00 €.

## Disputas

Una disputa (o contracargo) es cuando un alumno le dice a su banco que no reconoce un pago o que no recibió lo que compró, y el banco retira el dinero mientras lo revisa. **Las disputas corren siempre de tu cuenta, ganes o pierdas.**

1. Didacta te avisa **por correo y en tu panel** en cuanto Stripe le avisa, con la fecha límite para responder. Mientras se decide, esa venta queda retenida.
2. Si tienes pruebas de que el alumno recibió lo que compró (accesos, mensajes, factura), **envíaselas por correo**: Didacta las presenta ante Stripe en tu nombre.
3. Cuando el banco retira el dinero, se revierte de tu cuenta de cobros lo que te transfirieron por esa venta.

| Resultado | Qué asumes tú | El 5 % de Didacta |
| --- | --- | --- |
| **Pierdes** | El importe de la venta, la tasa de disputa de Stripe (hoy 20 € o su equivalente) y la comisión de Stripe de esa venta. | Didacta te lo devuelve: el dinero vuelve al alumno. |
| **Ganas** | La tasa de la disputa que Stripe no devuelva. Lo revertido se te devuelve. | No se devuelve: el alumno no recupera su dinero. |

## Lo que debas

Si después de un reembolso o una disputa Didacta te debe dinero a ti, o tú a Didacta:

- **Lo que debas se descuenta de tus próximos cobros**, en la misma moneda. Ves cada deuda en tu panel con su motivo y su importe pendiente.
- Si en **30 días** no hay cobros suficientes para cubrirla, o has dejado de usar Didacta Cloud, Didacta te pide el pago por correo con el detalle y tienes **otros 30 días** para pagarla por transferencia. Si no la pagas, puede reclamártela por las vías legales.
- Si Didacta te debe algo a ti (por ejemplo, una disputa que has ganado después de terminar el contrato), te lo paga en tu cuenta de cobros o, si ya no existe, por transferencia a la cuenta que indiques.

## Datos fiscales

Para que te lleguen los pagos, el aula te pide tus **datos fiscales** (en la sección **Cobros**, que en modo sencillo está en **Ventas → Ver detalles**, y como paso final de «Conecta tus cobros»). La ley obliga a las plataformas a recogerlos, y con ellos se emite la factura de la comisión.

| Dato | Notas |
| --- | --- |
| **¿Cómo vendes?** | Particular o autónomo, o empresa. |
| **Nombre o razón social** | |
| **País de residencia fiscal** | |
| **NIF** | Tu DNI o NIE con la letra, o el NIF de tu empresa; fuera de España, tu número de identificación fiscal. El aula comprueba el formato y la letra de control. |
| **NIF-IVA** (opcional) | Si lo tienes y eres de la UE, la factura va sin IVA y lo declaras tú. Se comprueba en VIES. |
| **Dirección fiscal** | |
| **Fecha de nacimiento** | Solo personas físicas: la pide la normativa de intercambio de información de plataformas digitales (DAC7). |

!!! warning "Sin datos fiscales, el dinero se queda retenido"
    Mientras no los completes, tus ventas se **cobran igualmente**, pero Didacta **no transfiere** el dinero a tu cuenta de cobros. Queda a salvo y te llega en cuanto los completas. Puedes publicar y vender sin ellos, pero el dinero no te llega.

Los datos se guardan **cifrados** y solo se usan para tus facturas y para lo que exige la normativa fiscal. Como operador de plataforma, Didacta puede tener que comunicar cada año a la Agencia Tributaria tus ventas y las comisiones cobradas (modelo 238). Si no entregas los datos tras dos recordatorios, la ley obliga a dejar de pagarte o a cerrar tu cuenta.

### Factura de la comisión

Cada mes Didacta te emite una factura de su comisión del mes anterior, con el IVA desglosado según tu caso (en la sección **Cobros**, tarjeta **Tus facturas**, descarga en PDF). Verás el régimen aplicado: IVA incluido, IVA español incluido, sin IVA por inversión del sujeto pasivo, sin IVA por estar fuera de la UE o sin IVA por Canarias, Ceuta o Melilla. Si una factura emitida se corrige, aparece una **rectificativa**. Más en [Precio y comisión](pricing.md#impuestos-sobre-la-comision).

## Tú eres el vendedor

El contrato de compra es **entre tú y tu alumno**: Didacta te presta la herramienta y cobra en tu nombre. Por eso:

- Tú decides el precio, lo que incluye cada curso o membresía, tus condiciones de venta y tu política de reembolsos, y las publicas en tu aula.
- Respondes ante tus alumnos del contenido y de que sea lo que anunciaste.
- Das a tus alumnos la información que exige la ley antes de comprar: quién eres, qué compran, el precio total con impuestos, cuánto dura y cómo se cancela una suscripción, y su derecho a desistir.
- Si vendes a consumidores de la UE, tienen **14 días para desistir**. En el contenido digital ese derecho se pierde si el alumno pide empezar antes de que acaben los 14 días, lo consiente de forma expresa y reconoce que lo pierde; la página de pago de Didacta recoge ese consentimiento cuando compra contenido digital.
- **Tú emites las facturas a tus alumnos** cuando la ley lo exija: el recibo de Stripe no es una factura.
- **Tú declaras y pagas los impuestos de tus ventas** (IVA, OSS…).

!!! warning "Impuestos: consulta a tu asesor"
    Esto es información, no asesoramiento fiscal. Las condiciones de uso para creadores explican, además, una advertencia sobre el IVA de los servicios electrónicos vendidos a través de plataformas: Didacta está diseñada para que el proveedor seas tú, pero no puede garantizar cómo calificará cada administración tributaria estas ventas. Léela en las [condiciones](https://didacta.io/es/legal/terms) (apartado 6).
