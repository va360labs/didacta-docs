# Tu aula

Qué puedes personalizar en tu aula de Didacta Cloud y qué no. En modo sencillo lo básico está en **Ajustes**; el resto, en **Ajustes → Todas las opciones avanzadas** (o en el [modo completo](../installation/visual-walkthrough.md#10-el-modo-sencillo)).

## Dirección

Tu aula vive en `https://<nombre-de-tu-academia>.didacta.io`. Esa dirección se deriva del nombre que pones al darte de alta: en minúsculas, sin acentos y con guiones en lugar de espacios y signos (por ejemplo, «Taller de Acuarela» da `taller-de-acuarela`). Si ya está cogida se le añade un número, y hay nombres reservados que no se conceden (`www`, `api`, `admin`, `login`…). La ves en la segunda pregunta del alta («Tu dirección»).

!!! note "Dominio propio"
    Hoy no se puede usar un dominio propio en Didacta Cloud. Sí puedes poner tu logo y tu color. Por eso el menú no tiene «Dominios propios» (tampoco en «Más» del modo sencillo) y, si abres `/admin/domains`, la pantalla dice «No disponible en Didacta Cloud» y te lleva a cambiar la dirección (desde la versión 0.1.0-beta.29).

### Cambiar la dirección

La cambias tú, sin pedírselo a nadie, en **Ajustes → «Dirección de tu academia»** (en modo completo, **Administración → Configuración → General**, `/admin/settings?tab=general`). Solo puede hacerlo quien administra el aula.

1. Escribe la **«Nueva dirección»**: solo la parte de delante, porque `.didacta.io` es fijo. Mientras escribes se pone en minúsculas, sin tildes y con guiones (por ejemplo, «Academía Pérez» da `academia-perez`) y se comprueba si está libre: «¡Está libre!», «Ya la usa otra academia. Prueba con otra.», «Esta dirección no se puede usar. Prueba con otra.» (nombres reservados) o «Usa solo letras sin tildes, números y guiones (entre 3 y 63).».
2. Pulsa **«Cambiar dirección»**. La confirmación te dice de cuál a cuál pasa y lo que viene a continuación; pulsa **«Sí, cambiar la dirección»**.
3. Te llevamos a la dirección nueva para que vuelvas a entrar.

Lo que conviene saber antes de cambiarla:

- **Hasta 3 cambios cada 30 días.** La tarjeta dice cuántos te quedan (por ejemplo, «Cambios que te quedan: 2 de 3 (cada 30 días).») y, si los has gastado, a partir de qué día podrás volver a cambiarla. Cuentan los cambios de los últimos 30 días.
- **La dirección anterior sigue siendo de tu aula** y ninguna otra academia puede quedársela: «Los enlaces antiguos seguirán funcionando: llevarán solos a la dirección nueva.» Las páginas llevan a la misma página en la dirección nueva (con su ruta y sus parámetros), así que sirven los enlaces de correos ya enviados, los marcadores y los que hayas publicado. Lo que va por `/api` (calendarios `.ics`, agentes MCP, webhooks) sigue respondiendo en la vieja. En los primeros minutos tras el cambio (hasta 5), la vieja aún puede abrir el aula sin redirigir. Si vuelves a una dirección que ya fue tuya, se reactiva.
- **Todo el mundo tiene que volver a entrar**: «Todo el mundo, tú también, tendrá que volver a entrar con su correo y su contraseña en la dirección nueva.» Las sesiones son de cada dirección y no pasan de una a otra.
- **Inicio de sesión único (SSO).** Si diste de alta la dirección del aula en un proveedor de inicio de sesión, cámbiala allí por la nueva. Por ejemplo, la «URL de callback (pegar en wp-config.php)» del [SSO desde WordPress](../modules/catalog/wp-sso.md) lleva la dirección del aula: copia la nueva de **Identidad (SSO)** y pégala en tu `wp-config.php`. Mientras, la vieja sigue respondiendo, pero no dependas de ella.
- **Tu web y tus redes**: «Si tienes la dirección en tu web o en tus redes, cámbiala allí cuando puedas.»
- **Lo que se pone al día solo**: los correos, las invitaciones de calendario y lo que ven los buscadores usan la dirección nueva desde el cambio, y la web de tu negocio que enseña Stripe se actualiza sola.
- **Un aula con dominio propio como dirección principal no la cambia desde aquí** («Tu academia usa su propio dominio (…), así que su dirección no se cambia desde aquí.»).

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
