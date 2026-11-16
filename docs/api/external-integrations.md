# Vender e integrar desde fuera de Didacta

Esta página es para cuando **la tienda, la web o el CRM viven fuera** y Didacta pone el aula: un WordPress que quiere dejar de pintar sus fichas con LearnDash, un CMS propio que cobra con su Stripe, un n8n que matricula al recibir un pedido.

Son tres APIs complementarias, todas con API key del tenant:

| | Para qué | Scopes |
|---|---|---|
| **`/inscribe`** — escritura | Dar y quitar acceso tras un pago que cobraste tú | `enrollments:write`, `courses:read` |
| **`/integrations`** — lectura | Pintar la ficha del curso con datos reales y saber si quien la mira ya compró | `courses:read`, `enrollments:read` |
| **`/integrations/orders`** — historial | Guardar en el perfil del alumno las compras hechas en tu tienda y leerlas de vuelta | `orders:write`, `orders:read` |

Y, en el sentido contrario, Didacta puede **avisarte** de lo que pasa en el aula con [webhooks salientes](webhooks.md#webhooks-salientes).

## Antes de nada: esto se llama desde tu servidor

**La API de Didacta no habilita CORS.** No es un descuido que se vaya a corregir: es la forma de la integración.

- Desde el navegador, las llamadas fallan.
- Y aunque no fallaran, no querrías hacerlas: **la API key da acceso a todo el tenant**. Publicarla en el JavaScript de tu web es entregarla.

Todo lo de esta página se llama **desde el backend** de tu sitio: el `functions.php` del WordPress, la route handler de tu CMS, el nodo HTTP de n8n.

!!! warning "El caudal no es el mismo por cada puerta"
    El límite de peticiones va **por organización** (100/min en Community, ajustable con `RATE_LIMIT_COMMUNITY_AUTH_PER_MIN`). Pero los endpoints **públicos** —el catálogo de venta sin autenticar— comparten **un solo cupo de 30/min para toda la instalación**, entre todas las organizaciones y todos los visitantes.

    Si tu web pinta fichas en cada visita, **usa los endpoints con API key**, no los públicos. Con los públicos compites con el tráfico anónimo de todo el servidor.

## Crear la clave

En **Administración → Claves API**, botón «Nueva clave». Elige solo los scopes que vayas a usar; el token en claro se enseña **una sola vez**.

```bash
curl -H "Authorization: ApiKey lmsk_..." \
     https://tu-aula.example.com/api/v1/integrations/courses
```

Fíjate en el esquema: `ApiKey`, no `Bearer`.

Los scopes que puede llevar una clave son un catálogo cerrado:

| Scope | Concede |
|---|---|
| `enrollments:write` | Matricular y dar de baja (`/inscribe`) |
| `enrollments:read` | Consultar el estado de cualquier email (`/integrations/learners/state`) |
| `courses:read` | Leer catálogo y grupos de acceso |
| `orders:write` | Guardar compras externas (`POST /integrations/orders`) |
| `orders:read` | Leer el historial de compra de cualquier email (`GET /integrations/learners/orders`) |
| `community:post` | Publicar en la comunidad (`POST /community-api/posts`) |
| `agent:*` | Los de la [puerta de agentes](mcp.md): no sirven para estas APIs |

### Por qué `enrollments:read` y `orders:read` van aparte

`enrollments:read` y `orders:read` permiten preguntar **por cualquier email de la organización**. Quien solo necesita pintar una ficha de venta no tiene por qué poder consultar a toda la base de alumnos, ni al revés. Si tu integración solo lee catálogo, dale `courses:read` y nada más. Lo mismo entre `orders:write` y `orders:read`: la tienda que guarda pedidos no tiene por qué poder leer los de todo el aula.

## Leer: pintar la ficha fuera de Didacta

### Encontrar el curso

```bash
# Por slug
GET /api/v1/integrations/courses?slug=curso-de-claude-code

# O por el id que tenía en el LMS de origen, si lo importaste
GET /api/v1/integrations/courses?externalId=1234&externalSource=learndash
```

El filtro por `externalId` + `externalSource` es el que evita copiar UUIDs a mano: si los cursos entraron con el importador de LearnDash, cada uno recuerda de qué post de WordPress viene, y tu plugin puede resolver el mapeo solo.

### La ficha completa

```bash
GET /api/v1/integrations/courses/{courseId}
```

Devuelve todo lo que necesita una página de venta: descripción, imagen, vídeo destacado, formador, si emite certificado, totales (módulos, clases, minutos), **el temario módulo a módulo** y, si el curso tiene precio en Didacta, sus **opciones de compra** con importe y precio tachado.

!!! info "Nunca devuelve el contenido de una lección"
    El temario se enseña para vender; la clase se da en Didacta. `content` no sale de esta API bajo ninguna circunstancia, ni con todos los scopes.

Una lección puede venir con `scheduled: true`: existe en el temario pero tiene fecha de publicación futura. Píntala como «próximamente», no la escondas.

### ¿Quien mira esto ya lo compró?

```bash
GET /api/v1/integrations/learners/state?email=ana@ejemplo.com&courseId={uuid}
```

Es la pregunta que se hace toda ficha de curso externa. La respuesta decide qué botón pintas:

| Respuesta | Qué significa | Qué pintar |
|---|---|---|
| `known: false` | Ese email no es de nadie en tu organización | Modo venta |
| `hasAccess: true` | Está matriculado y puede entrar | «Continuar por donde ibas» con `nextLesson.url` |
| `hasAccess: false` con `status: "PAUSED"` | **Alumno suspendido**, típicamente por impago | Ni venta ni acceso: avísale de que regularice |

Ese último caso es el que se hace mal. `PAUSED` **no** es «nunca compró»: su progreso está intacto y volverá en cuanto se resuelva el pago. Si lo tratas como visitante, le ofreces comprar algo que ya pagó.

## Escribir: dar acceso tras cobrar tú

```bash
curl -X POST https://tu-aula.example.com/api/v1/inscribe \
  -H "Authorization: ApiKey lmsk_..." \
  -H "Content-Type: application/json" \
  -d '{
        "email": "ana@ejemplo.com",
        "name": "Ana Pérez",
        "courseIds": ["3f4b2c10-1a2b-4c3d-9e8f-0a1b2c3d4e5f"],
        "externalRef": "pedido_12345"
      }'
```

Crea el usuario si no existe (recibe por email una contraseña temporal que debe cambiar al entrar) y lo matricula. **Es idempotente**: repetir la misma llamada no duplica nada, así que puedes reintentar sin miedo desde tu cola de webhooks.

### Packs: un grupo de acceso, no N matriculaciones

Si vendes un pack de varios cursos, **no mandes N `courseIds`**. Crea un grupo de acceso de tipo `MULTI_COURSE` en Didacta y matricula contra él:

```json
{ "email": "ana@ejemplo.com", "accessGroupIds": ["<uuid-del-pack>"] }
```

La diferencia importa cuando el pack cambia: con un grupo, quien ya compró ve el curso nuevo sin que tu tienda haga nada. Con matrículas sueltas, el acceso se congeló el día de la compra.

`GET /inscribe/access-groups` te da los grupos con su `kind` y cuántos cursos otorgan.

### Reembolsos: cablea la baja el primer día

```bash
POST /api/v1/inscribe/revoke
{ "email": "ana@ejemplo.com", "courseIds": ["..."], "reason": "refund" }
```

Es la mitad que todo el mundo deja para después y nadie vuelve a hacer. Sin ella, quien pide la devolución conserva el curso.

Es deliberadamente conservador: **solo cancela matrículas de origen API**. Si esa persona además es miembro o tiene el curso por un grupo de acceso, ese acceso **no se toca** — le has devuelto su compra, no le retiras lo que ya tenía por otra vía.

## Guardar el historial de compra en el perfil

Si la tienda cobra con su pasarela y da el acceso con `/inscribe`, puede además **dejar el pedido en el perfil del alumno**: así el historial vive en un solo sitio y las dos pantallas —la del aula y la de la tienda— dicen lo mismo.

```bash
curl -X POST https://tu-aula.example.com/api/v1/integrations/orders \
  -H "Authorization: ApiKey lmsk_..." \
  -H "Content-Type: application/json" \
  -d '{
        "email": "ana@ejemplo.com",
        "source": "tienda.example.com",
        "reference": "PED-260818-0042",
        "status": "PAID",
        "amountCents": 4770,
        "currency": "eur",
        "placedAt": "2026-11-16T10:15:00Z",
        "lines": [{ "name": "Curso de ejemplo", "amountCents": 4770 }],
        "orderUrl": "https://tienda.example.com/cuenta/pedidos/42"
      }'
```

- **Es idempotente por `(source, reference)`**: un webhook de tu tienda se reintenta, y el reintento actualiza la fila en vez de duplicarla. `source` identifica la tienda (puede haber más de una) y `reference` es el número de pedido **en tu tienda**.
- Obligatorios: `email`, `source`, `reference`, `amountCents` (entero, en la unidad mínima de la moneda) y `placedAt`. `status` es `PAID` (por defecto), `REFUNDED`, `PARTIALLY_REFUNDED` o `CANCELLED`; `currency` por defecto `eur`.
- **Omitir no borra.** `invoice`, `orderUrl` y `refundedAt` solo se escriben si vienen, de modo que puedes volver media hora después con el número de factura sin reenviar el pedido entero. Para **retirar** una factura (anulada, rectificada) hay que mandar `"invoice": null` explícitamente.
- `invoice` lleva `number`, `issuedAt` y `url` (al menos el número o el enlace). **Didacta no emite facturas ni numera series fiscales**: guarda el número, la fecha y un enlace al PDF que sirve quien la emitió.
- Si el comprador todavía no existe en el aula, el pedido se guarda igual y se localiza por su email.

Para leerlo de vuelta —por ejemplo, para la zona de cliente de tu tienda—:

```bash
GET /api/v1/integrations/learners/orders?email=ana@ejemplo.com&source=tienda.example.com&limit=50
```

Devuelve `known`, `userId` y `orders` (de la más reciente a la más antigua; `limit` de 1 a 200). **`known: false` no implica lista vacía**: un pedido puede haber llegado antes de que `/inscribe` creara la cuenta. La propia persona ve esas compras en su perfil (`GET /api/v1/me/purchases`, con su sesión).

## Enterarte de lo que pasa en Didacta

Si además de escribir quieres que Didacta te avise, configura un **webhook saliente** en **Administración → Webhooks** (o por API). Los eventos útiles para una tienda externa:

| Evento | Cuándo |
|---|---|
| `billing.order.completed` | Se pagó un curso suelto **en Didacta** |
| `billing.order.refunded` | Se devolvió esa compra → retira el acceso en tu sistema |
| `billing.order.failed` | El pago se cayó (o el diferido no llegó a cobrarse) |
| `learning.enrollment.created` | Alguien quedó matriculado, por la vía que sea |
| `learning.enrollment.paused` | Suspendido por impago: conserva el progreso, pierde el acceso |
| `learning.enrollment.resumed` | Recuperó el acceso tras regularizar |
| `learning.enrollment.cancelled` | Baja de la matrícula (reembolso, cancelación, salida de un grupo) |
| `learning.course.completed` | Terminó el curso (para tu email de felicitación, tu CRM…) |
| `subscriptions.invoice.paid` | **Renovación cobrada** — el único evento que distingue «sigue pagando» de «se dio de alta» |
| `subscriptions.invoice.payment_failed` | Falló el cobro de la cuota |
| `subscriptions.subscription.past_due` | Falló un cobro y sigue dentro (periodo de gracia): la ventana para recuperarlo |
| `subscriptions.subscription.canceled` | Fin de la suscripción |

El catálogo completo, el cuerpo, las cabeceras y los reintentos están en [Webhooks](webhooks.md#webhooks-salientes). Lo que importa aquí:

- Además del `data` propio del evento, llega **`learner` con el email y el nombre de la persona**: es lo que permite cruzarlo con tu CRM, porque `data` solo trae el `userId` y no hay endpoint que lo traduzca.
- **Deduplica por la cabecera `X-Didacta-Delivery`**: ante un reintento el mismo evento puede llegar dos veces, y esa cabecera vale lo mismo en los dos envíos.
- ⚠️ **En Community el POST NO va firmado** (la firma HMAC es de Enterprise) y, si tu endpoint está caído, tras un único reintento el aviso se pierde. Mete un token en la URL del endpoint y compruébalo, y no hagas que tu integración dependa **solo** de los webhooks: reconcilia de vez en cuando con `GET /integrations/learners/state`.

## Los tres errores que se cometen siempre

1. **Llamar desde el navegador.** No hay CORS, y la clave quedaría publicada. Siempre desde tu servidor.
2. **Tratar `PAUSED` como «no ha comprado».** Es un alumno suspendido con su progreso intacto.
3. **No cablear el reembolso.** `POST /inscribe` sin su `revoke` es media integración.

## Ver también

- [Autenticación](authentication.md) — crear y revocar API keys, scopes disponibles.
- [Webhooks](webhooks.md) — los webhooks salientes (catálogo de eventos, firma, reintentos) y los entrantes de Stripe, Zoom y WooCommerce.
- [Agentes de IA y servidor MCP](mcp.md) — la otra puerta con clave de API: para que un agente cree cursos, directos y publicaciones.
- [Referencia: núcleo y transversales](reference/core.md) — la tabla completa de rutas.
- [Grupos de acceso](../modules/catalog/access-groups.md) — cómo se monta un pack.
- [SSO desde WordPress](../modules/catalog/wp-sso.md) — para que el comprador entre al aula sin volver a identificarse. El mecanismo es un JWT firmado con un secreto compartido: **sirve para cualquier sitio externo**, no solo WordPress.
