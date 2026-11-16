# Tu aula

Qué puedes personalizar en tu aula de Didacta Cloud y qué no. En modo sencillo lo básico está en **Ajustes**; el resto, en **Ajustes → Todas las opciones avanzadas** (o en el [modo completo](../installation/visual-walkthrough.md#10-el-modo-sencillo)).

## Dirección

Tu aula vive en `https://<nombre-de-tu-academia>.didacta.io`. Esa dirección se deriva del nombre que pones al darte de alta: en minúsculas, sin acentos y con guiones en lugar de espacios y signos (por ejemplo, «Taller de Acuarela» da `taller-de-acuarela`). Si ya está cogida se le añade un número, y hay nombres reservados que no se conceden (`www`, `api`, `admin`, `login`…). La ves en la segunda pregunta del alta («Tu dirección»).

!!! note "Dominio propio"
    Hoy no se puede usar un dominio propio en Didacta Cloud. Sí puedes poner tu logo y tu color.

## Logo y color

En **Ajustes → Logo y color** (en modo completo, **Administración → Branding**) subes tu logo y eliges tu color. Lo ven tus alumnos en la cabecera, en tu página de venta y en el pago.

- **Sin logo**, el aula usa **tus iniciales** sobre tu color.
- El pago se hace dentro de tu página de venta, con tu logo (o tus iniciales) y tu color. Los recibos y correos que reciben tus compradores llevan tu marca y su texto lo editas tú en **Administración → Emails**.
- El pie del aula lleva «Powered by Didacta». Ocultar la marca Didacta es la capability Enterprise de [marca blanca](../configuration/branding.md#white-label-completo-enterprise).

## Idioma

Tu aula puede ir en **español, inglés o portugués de Brasil**. El del aula es:

- el de tu página de venta y tu catálogo;
- el de los correos a quien todavía no ha elegido idioma;
- el de los textos que el sistema escribe para todos (por ejemplo, el anuncio de un directo en la comunidad).

Nace en el idioma desde el que te diste de alta (`/es`, `/en` o `/pt-br`). Para cambiarlo: **Ajustes → Idioma de la academia** (en modo completo, **Administración → Configuración → General**). **Cada alumno puede elegir el suyo** en su perfil.

## Correo

Los correos de tu aula (avisos, recuperar contraseña, notificaciones, recibos de compra) salen por el **servicio de correo de la plataforma**, con el **nombre de tu academia** como remitente. Tú no configuras nada para que lleguen.

| Qué | Dónde |
| --- | --- |
| **Adónde van las respuestas.** Si un alumno contesta a un correo, la respuesta llega al «Correo para respuestas» que fijes. Si no pones ninguno, llega al correo de la persona que administra el aula. | **Administración → Configuración → Notificaciones** (`/admin/settings?tab=notifications`) |
| **Enviar desde tu propio servicio de correo.** Si prefieres que salgan por tu SMTP, lo conectas en la misma pestaña y envías un correo de prueba. | Misma pestaña. Ver [Email](../configuration/email.md) |
| **El texto de cada correo.** Cambias el asunto y el cuerpo de los correos que envía la plataforma. | **Administración → Emails** |

!!! tip "Pon un «Correo para respuestas»"
    Es un solo campo y evita que las respuestas de tus alumnos acaben en un buzón que no miras.

## Qué no se puede cambiar en Cloud

Cloud no expone la configuración de la instancia: no hay variables de entorno, ni almacenamiento propio, ni acceso a la base de datos, porque no operas el servidor. La lista completa está en [Límites y diferencias con Community](cloud-vs-community.md).
