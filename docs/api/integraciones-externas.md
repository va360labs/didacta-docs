# Vender e integrar desde fuera de Didacta

Esta página es para cuando **la tienda, la web o el CRM viven fuera** y Didacta pone el aula: un WordPress que quiere dejar de pintar sus fichas con LearnDash, un CMS propio que cobra con su Stripe, un n8n que matricula al recibir un pedido.

Son dos APIs complementarias, las dos con API key del tenant:

| | Para qué | Scopes |
|---|---|---|
| **`/inscribe`** — escritura | Dar y quitar acceso tras un pago que cobraste tú | `enrollments:write`, `courses:read` |
| **`/integrations`** — lectura | Pintar la ficha del curso con datos reales y saber si quien la mira ya compró | `courses:read`, `enrollments:read` |

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

### Por qué `enrollments:read` va aparte

`enrollments:read` permite preguntar **por cualquier email de la organización**. Quien solo necesita pintar una ficha de venta no tiene por qué poder consultar a toda la base de alumnos, ni al revés. Si tu integración solo lee catálogo, dale `courses:read` y nada más.

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

## Enterarte de lo que pasa en Didacta

Si además de escribir quieres que Didacta te avise, configura un **webhook saliente** en Administración → Webhooks. Los eventos útiles para una tienda externa:

| Evento | Cuándo |
|---|---|
| `billing.order.completed` | Se pagó un curso suelto **en Didacta** |
| `billing.order.refunded` | Se devolvió esa compra → retira el acceso en tu sistema |
| `learning.enrollment.created` | Alguien quedó matriculado, por la vía que sea |
| `learning.course.completed` | Terminó el curso (para tu email de felicitación, tu CRM…) |

El endpoint recibe un POST firmado. **Deduplica por la cabecera `X-Didacta-Delivery`**: ante un reintento, el mismo evento puede llegar dos veces.

## Los tres errores que se cometen siempre

1. **Llamar desde el navegador.** No hay CORS, y la clave quedaría publicada. Siempre desde tu servidor.
2. **Tratar `PAUSED` como «no ha comprado».** Es un alumno suspendido con su progreso intacto.
3. **No cablear el reembolso.** `POST /inscribe` sin su `revoke` es media integración.

## Ver también

- [Autenticación](autenticacion.md) — crear y revocar API keys, scopes disponibles.
- [Referencia: núcleo y transversales](referencia/nucleo.md) — la tabla completa de rutas.
- [Grupos de acceso](../modulos/catalogo/access-groups.md) — cómo se monta un pack.
- [SSO desde WordPress](../modulos/catalogo/wp-sso.md) — para que el comprador entre al aula sin volver a identificarse. El mecanismo es un JWT firmado con un secreto compartido: **sirve para cualquier sitio externo**, no solo WordPress.
