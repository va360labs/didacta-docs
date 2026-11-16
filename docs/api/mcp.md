# Agentes de IA y servidor MCP

Cada aula de Didacta trae un **servidor MCP** (Model Context Protocol) en `POST /api/v1/mcp`. Es la «puerta de agentes»: un agente de IA (Claude Code, Codex, Hermes, Grok, Cursor, GitHub Copilot en VS Code…) se conecta con una clave propia y hace por ti, dentro de tu aula, lo que hoy haces a mano: **crear y editar cursos y lecciones, programar clases en directo y publicar en la comunidad**.

No es una segunda API de negocio: cada herramienta llama al mismo servicio de módulo que usa el panel (mismas validaciones, mismo saneado de texto) y actúa **en nombre de la persona administradora que creó la clave**.

!!! info "Qué puede hacer y qué no"
    **Puede** crear y editar cursos y lecciones, programar clases en directo y publicar en la comunidad.

    **No puede**, y no hay scope que lo permita: borrar nada, dar de alta o invitar personas, poner precios, publicar un curso a la venta ni avisar por correo a todos los miembros. Los cursos que crea un agente quedan en **borrador**; los publicas tú, con su precio, desde «Empieza aquí». En cambio, las clases en directo, las publicaciones de la comunidad y las lecciones añadidas a un curso **ya publicado** se ven al momento: confirma con el agente antes de pedirle esas cosas.

## Crear la clave de un agente

Hay dos caminos. Los dos exigen ser **administrador** del aula (`tenant_admin` o `super_admin`).

=== "Desde el panel (lo normal)"

    En las pantallas de gestión del aula —**Cursos** (`/studio/courses`), el editor de un curso y **Directos** (`/studio/live`)— aparece el panel **«Hazlo más rápido con tu agente de IA»**:

    1. **Crea tu clave de conexión**: pulsa «Crear clave». El token (`lmsk_…`) se enseña **una sola vez**.
    2. **Conecta tu agente**: elige su pestaña y copia el comando o el bloque de configuración, que ya lleva la URL de tu aula y tu clave.

    Por debajo, el panel llama a `POST /api/v1/agent/keys`. Esa ruta solo acepta una **sesión de persona** (no una clave de API: un agente no puede acuñarse claves nuevas), el cuerpo solo admite un `name` opcional y **los scopes los fija el servidor**: siempre los cuatro `agent:*`.

=== "Por API"

    Con una sesión de administrador puedes crear la clave en `POST /api/v1/admin/api-keys` eligiendo tú los scopes `agent:*` (y, si quieres, `expiresAt`):

    ```bash
    curl -X POST https://tu-aula.example.com/api/v1/admin/api-keys \
      -H "Authorization: Bearer <accessToken>" \
      -H "Content-Type: application/json" \
      -d '{"name": "Agente de cursos", "scopes": ["agent:courses:read", "agent:courses:write"]}'
    ```

    La respuesta lleva el `token` en claro, una sola vez.

Las claves de agente se listan y se **revocan** en **Administración → Claves API**, como cualquier otra. En la base solo se guarda el SHA-256 del token; la lista muestra la fecha de creación y el último uso.

!!! warning "La clave actúa como tú"
    Lo que el agente crea queda a nombre de la persona que generó la clave. Si a esa persona le quitan el rol de administrador (o la cuenta deja de estar activa), **la clave deja de servir en ese mismo momento**.

## Endpoint y transporte

| | |
|---|---|
| **URL** | `https://tu-aula.example.com/api/v1/mcp` (el dominio de tu aula) |
| **Transporte** | «Streamable HTTP» de MCP, **sin estado**: cada `POST` lleva un mensaje JSON-RPC 2.0 y se responde con `application/json` |
| **Sesiones y SSE** | No hay `Mcp-Session-Id` ni flujo SSE: `GET` y `DELETE` responden `405` |
| **Notificaciones** (mensajes sin `id`) | `202` sin cuerpo |
| **Lotes** (array JSON-RPC) | Se aceptan |
| **Versiones de protocolo** | `2025-11-25`, `2025-06-18`, `2025-03-26`, `2024-11-05` (si el cliente pide otra, responde `2025-11-25`) |
| **Métodos** | `initialize`, `ping`, `tools/list`, `tools/call`. `resources/list`, `resources/templates/list` y `prompts/list` devuelven listas vacías; cualquier otro método, `-32601` |
| **Cabecera `MCP-Protocol-Version`** | No es obligatoria |

`initialize` devuelve `serverInfo` con el nombre del aula (`Didacta · <nombre>`) y unas `instructions` para el modelo (en inglés, citando los nombres de pantalla en el idioma del aula): qué puede hacer, que los cursos nuevos son borradores y que debe escribir el contenido en el idioma que use el propietario.

!!! note "Sin CORS, igual que el resto de la API"
    Los clientes MCP se conectan desde tu máquina o desde un servidor, no desde una página web. Ver [Convenciones](conventions.md#otras-convenciones).

## Autenticación

La clave va en la cabecera `Authorization`, con cualquiera de estos dos esquemas:

```http
Authorization: Bearer lmsk_…
Authorization: ApiKey lmsk_…
```

`Bearer` es el que ofrecen todos los clientes MCP con su opción de «token»; `ApiKey` es el del resto de la API de Didacta. No se admite la clave por query string ni por otra cabecera, y **un JWT de sesión no entra**: esta puerta es solo para claves propias, revocables y con scopes.

| Código HTTP | `code` | Cuándo |
|---|---|---|
| 401 | `AGENT_KEY_MISSING` | Falta la cabecera o la clave no empieza por `lmsk_` |
| 401 | `AGENT_KEY_INVALID` | La clave no existe, ha caducado o se ha revocado |
| 403 | `AGENT_KEY_NOT_AGENT` | La clave es válida pero no tiene ningún scope `agent:*` |
| 403 | `AGENT_KEY_OTHER_ACADEMY` | La petición llega por el dominio de otra aula |

### Una clave, un aula

El aula sale **de la clave y de nada más**: ninguna cabecera, parámetro ni argumento de herramienta puede cambiarla (las herramientas ni siquiera tienen argumento de aula). El dominio solo sirve para **rechazar**: pegar la clave de un aula en la URL de otra devuelve `AGENT_KEY_OTHER_ACADEMY` en vez de escribir donde no toca. Un `courseId` de otra aula simplemente no existe para esta.

## Scopes

Son verbos de producto, no espejos de endpoints:

| Scope | Concede |
|---|---|
| `agent:courses:read` | Listar cursos y leer un curso con sus lecciones |
| `agent:courses:write` | Crear cursos y lecciones; renombrar y describir cursos; editar lecciones |
| `agent:live:write` | Ver las próximas clases en directo y programar una nueva |
| `agent:community:post` | Ver los espacios de la comunidad y publicar en uno |

`tools/list` devuelve **solo** las herramientas que los scopes de la clave permiten **y** cuyo módulo está activo en el aula (`mod.courses`, `mod.zoom-live`, `mod.community`): el agente nunca ve algo que luego se le vaya a negar. Los demás scopes de API (`enrollments:*`, `courses:read`, `orders:*`, `community:post`) son de otras APIs y no abren esta puerta; ver [Vender e integrar desde fuera](external-integrations.md).

## Herramientas

Son 10. Los argumentos se validan siempre en el servidor; un campo desconocido es un error (`additionalProperties: false`). Las descripciones que lee el modelo están en inglés; lo que el agente **crea** va en el idioma en que se lo pidas.

| Herramienta | Scope | Módulo | Escribe |
|---|---|---|---|
| `list_courses` | `agent:courses:read` | `mod.courses` | No |
| `get_course` | `agent:courses:read` | `mod.courses` | No |
| `create_course` | `agent:courses:write` | `mod.courses` | Sí |
| `update_course` | `agent:courses:write` | `mod.courses` | Sí |
| `create_lesson` | `agent:courses:write` | `mod.courses` | Sí |
| `update_lesson` | `agent:courses:write` | `mod.courses` | Sí |
| `list_live_classes` | `agent:live:write` | `mod.zoom-live` | No |
| `schedule_live_class` | `agent:live:write` | `mod.zoom-live` | Sí |
| `list_community_spaces` | `agent:community:post` | `mod.community` | No |
| `create_community_post` | `agent:community:post` | `mod.community` | Sí |

Las herramientas de solo lectura se anuncian con `readOnlyHint: true`; ninguna lleva `destructiveHint`, porque ninguna borra.

### Cursos y lecciones

**`list_courses`** — Lista los cursos del aula con `id`, `title`, `status` (`DRAFT`, `PUBLISHED`, `ARCHIVED`), número de lecciones y `editPath`. Devuelve como máximo 100.

| Parámetro | Tipo | Notas |
|---|---|---|
| `status` | enum, opcional | Solo cursos en ese estado |

**`get_course`** — Un curso con sus lecciones en orden: `id`, `number`, `title`, `type` y `hasContent` (si la lección ya tiene contenido).

| Parámetro | Tipo | Notas |
|---|---|---|
| `courseId` | uuid, obligatorio | |

**`create_course`** — Crea un curso **en borrador**. El enlace web sale del título (si ya existe, prueba `-2`, `-3`… hasta 20 veces); el idioma del curso es el del perfil de quien creó la clave.

| Parámetro | Tipo | Notas |
|---|---|---|
| `title` | texto 1–160, obligatorio | |
| `description` | texto ≤ 2000, opcional | Texto plano |

**`update_course`** — Cambia título y/o descripción. No toca el estado ni el precio.

| Parámetro | Tipo | Notas |
|---|---|---|
| `courseId` | uuid, obligatorio | |
| `title` | texto 1–160 | Al menos uno de `title` o `description` |
| `description` | texto ≤ 2000 | |

**`create_lesson`** — Añade una lección **al final** del curso (en su última sección; si el curso no tiene ninguna, la crea con el nombre por defecto del idioma del propietario: «Lecciones», «Lessons», «Aulas»). Se llama una vez por lección, en orden, y los títulos no se numeran: el aula numera sola.

| Parámetro | Tipo | Notas |
|---|---|---|
| `courseId` | uuid, obligatorio | |
| `title` | texto 1–160, obligatorio | |
| `type` | `text` \| `video`, obligatorio | |
| `text` | texto ≤ 100 000 | Solo con `type: "text"`. Se muestra como **texto plano** (los saltos de línea se conservan; ni HTML ni Markdown se interpretan) |
| `videoUrl` | URL `https://` ≤ 2000 | Solo con `type: "video"` (YouTube, Vimeo, Bunny…) |

**`update_lesson`** — Cambia título, texto (lecciones de texto) o enlace de vídeo (lecciones de vídeo). El tipo no cambia.

| Parámetro | Tipo | Notas |
|---|---|---|
| `lessonId` | uuid, obligatorio | |
| `title` | texto 1–160 | Al menos uno de `title`, `text` o `videoUrl` |
| `text` | texto ≤ 100 000 | Error `AGENT_WRONG_TYPE` si la lección no es de texto |
| `videoUrl` | URL `https://` | Error `AGENT_WRONG_TYPE` si la lección no es de vídeo |

### Clases en directo

**`list_live_classes`** — Las próximas clases (sin las canceladas), ordenadas por fecha, hasta 20: `id`, `title`, `startTime`, `timezone`, `durationMinutes`, `status` y `path`. Sin parámetros.

**`schedule_live_class`** — Programa una clase a la que el alumnado entra con **el enlace de videollamada que ya uses** (Zoom, Google Meet, Teams, Jitsi…). Con enlace propio, esta herramienta no habla nunca con la API de Zoom.

| Parámetro | Tipo | Notas |
|---|---|---|
| `title` | texto 1–200, obligatorio | |
| `startTime` | fecha-hora ISO 8601 **con desfase**, obligatorio | `2026-11-20T19:00:00+01:00` |
| `durationMinutes` | entero 15–480, obligatorio | |
| `timezone` | zona IANA, obligatorio | `Europe/Madrid` |
| `joinUrl` | URL `https://`, obligatorio | Enlace de la videollamada |
| `description` | texto ≤ 2000 | |
| `courseId` | uuid | Vincula la clase a un curso |
| `announce` | booleano | Con `true`, además se anuncia en la comunidad |

La clase queda con el correo del propietario de la clave como anfitrión y **se ve al momento**.

### Comunidad

**`list_community_spaces`** — Los espacios de la comunidad (`slug`, `title`). Sin parámetros.

**`create_community_post`** — Publica una entrada firmada por el propietario de la clave. **No envía correo a los miembros** y no se marca como importante.

| Parámetro | Tipo | Notas |
|---|---|---|
| `title` | texto 3–200, obligatorio | |
| `body` | texto 1–10 000, obligatorio | |
| `space` | slug (`[a-z0-9-]`, ≤ 60) | Si se omite, va a `general`. Un espacio que no existe da `AGENT_SPACE_UNKNOWN` con la lista de válidos |

Si el propietario de la clave tiene una sanción de comunidad que le impide publicar, su agente tampoco puede (`AGENT_OWNER_RESTRICTED`).

## Llamar a la puerta a mano

Útil para probar la clave:

```bash
curl -X POST https://tu-aula.example.com/api/v1/mcp \
  -H "Authorization: Bearer lmsk_…" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}'
```

Y una llamada a una herramienta:

```bash
curl -X POST https://tu-aula.example.com/api/v1/mcp \
  -H "Authorization: Bearer lmsk_…" \
  -H "Content-Type: application/json" \
  -d '{
        "jsonrpc": "2.0",
        "id": 2,
        "method": "tools/call",
        "params": {
          "name": "create_course",
          "arguments": { "title": "Introducción a la fotografía", "description": "Cuatro semanas para dominar tu cámara." }
        }
      }'
```

El resultado de una herramienta llega como un bloque `content` de tipo `text` con el JSON del resultado (para `create_course`: `id`, `title`, `status`, `editPath` y una `note`). Los fallos **de la herramienta** no son errores de protocolo: vuelven como resultado con `isError: true` y el texto `mensaje (CÓDIGO)`, para que el agente lo lea y corrija o se lo cuente a la persona.

| `code` | Significado |
|---|---|
| `AGENT_TOOL_UNKNOWN` | La herramienta no existe |
| `AGENT_SCOPE_MISSING` | La clave no tiene el scope que pide la herramienta |
| `AGENT_MODULE_OFF` | El módulo de la herramienta está apagado en esta aula |
| `AGENT_RATE_LIMITED` | Límite de llamadas superado (ver abajo) |
| `AGENT_INVALID_ARGUMENTS` | Argumentos que no cumplen el esquema |
| `AGENT_OWNER_GONE` / `AGENT_OWNER_NOT_ADMIN` | Quien creó la clave ya no está activo o ya no es administrador |
| `AGENT_NOT_FOUND`, `AGENT_WRONG_TYPE`, `AGENT_COURSE_SLUG`, `AGENT_SPACE_UNKNOWN`, `AGENT_OWNER_RESTRICTED` | Fallos de negocio de cada herramienta (ver arriba) |
| `AGENT_INTERNAL` | Error inesperado; no se filtran detalles internos |

Los errores propios de los módulos (cursos, clases en directo) mantienen su código.

## Límites

- **Por clave**: solo cuentan las llamadas `tools/call`. Por defecto **60 por minuto** y **600 por hora**; se ajustan con `AGENT_TOOL_CALLS_PER_MIN` y `AGENT_TOOL_CALLS_PER_HOUR` (ver [Variables de entorno](../configuration/environment-variables.md)). Al superarlos, la herramienta devuelve `AGENT_RATE_LIMITED` con los segundos que hay que esperar. Con Redis el contador se comparte entre réplicas; sin Redis cuenta en memoria del proceso (no deja pasar todo).
- **Por aula**: además rige el límite general de peticiones de la API, que comparten todas las claves del aula.
- **Tamaños**: ver la tabla de cada herramienta. `list_courses` devuelve hasta 100 cursos y `list_live_classes` hasta 20 clases.

## Configurar un cliente MCP

El panel genera estos bloques ya rellenos con la URL de tu aula y tu clave; aquí van con valores de ejemplo. `nombre-del-aula` es el nombre con el que el cliente guarda la conexión (el panel usa el identificador del aula —su *slug*— en minúsculas y con guiones).

=== "Claude Code"

    ```bash
    claude mcp add --transport http --scope user nombre-del-aula \
      https://tu-aula.example.com/api/v1/mcp \
      --header "Authorization: Bearer lmsk_TU_CLAVE"

    claude mcp list   # para comprobar que conectó
    ```

=== "Codex"

    En `~/.codex/config.toml`:

    ```toml
    [mcp_servers.nombre-del-aula]
    url = "https://tu-aula.example.com/api/v1/mcp"
    http_headers = { "Authorization" = "Bearer lmsk_TU_CLAVE" }
    ```

    Comprobar: `codex mcp list`.

=== "Hermes"

    En `~/.hermes/config.yaml`:

    ```yaml
    mcp_servers:
      nombre-del-aula:
        url: "https://tu-aula.example.com/api/v1/mcp"
        headers:
          Authorization: "Bearer lmsk_TU_CLAVE"
    ```

    Comprobar: `hermes mcp test nombre-del-aula`.

=== "Grok"

    Con Grok Build, en el terminal:

    ```bash
    grok mcp add --transport http nombre-del-aula \
      https://tu-aula.example.com/api/v1/mcp \
      --header "Authorization: Bearer lmsk_TU_CLAVE"
    ```

    En la app de Grok: *Connect apps → Add a custom connector → Remote HTTPS*, con la URL del servidor y la cabecera `Authorization: Bearer lmsk_TU_CLAVE`.

=== "Cursor"

    En `~/.cursor/mcp.json` (y actívalo en los ajustes de MCP de Cursor):

    ```json
    {
      "mcpServers": {
        "nombre-del-aula": {
          "url": "https://tu-aula.example.com/api/v1/mcp",
          "headers": { "Authorization": "Bearer lmsk_TU_CLAVE" }
        }
      }
    }
    ```

=== "VS Code"

    En `.mcp.json` de tu proyecto (GitHub Copilot en modo agente):

    ```json
    {
      "mcpServers": {
        "nombre-del-aula": {
          "type": "http",
          "url": "https://tu-aula.example.com/api/v1/mcp",
          "headers": { "Authorization": "Bearer lmsk_TU_CLAVE" }
        }
      }
    }
    ```

!!! note "Solo clientes que mandan una cabecera fija"
    La puerta se autentica con una clave en la cabecera `Authorization`. Un cliente que solo admita OAuth no puede conectarse (es el caso de ChatGPT).

Una vez conectado, pídele algo concreto: *«Crea un curso llamado "Introducción a la fotografía" con 6 lecciones de texto a partir de estas notas…»* o *«Programa un directo el martes que viene a las 19:00 (mi hora) con este enlace de videollamada…»*.

## Seguridad y auditoría

- **Aislamiento entre aulas**: el aula sale de la fila de la clave; ver [arriba](#una-clave-un-aula). Además cada llamada corre dentro del contexto de ese aula, de modo que la seguridad por filas de la base de datos (RLS) actúa como segunda cerradura.
- **Mínimo privilegio**: los scopes recortan qué herramientas existen para la clave, y lo que ningún scope concede (borrar, precios, publicar a la venta, invitar, avisar por correo a todos) **no existe** en esta puerta.
- **Clave por agente y revocable**: crea una por agente o por persona y revócala en **Administración → Claves API** al instante. Se guarda solo el hash del token; el token en claro no se puede recuperar.
- **Auditoría**: toda herramienta que escribe deja una entrada en el registro de auditoría (**Administración → Auditoría**) con el propietario de la clave como actor, la acción `agent.<herramienta>` (`agent.create_course`, `agent.schedule_live_class`…), el recurso afectado y en `metadata` el `apiKeyId` y `via: "mcp"`. Las herramientas de solo lectura no escriben entradas. La creación de la clave queda registrada como `admin.api_key.created` (con `purpose: "agent"` si salió del panel).
- **Sin licencia especial**: la puerta de agentes no está ligada a ninguna capability de licencia (ninguna de sus rutas responde `402`), así que está disponible en todas las ediciones.

## Ver también

- [Autenticación](authentication.md) — API keys, scopes y sesiones.
- [Vender e integrar desde fuera](external-integrations.md) — las APIs de API key para tiendas y CRM.
- [Cursos y catálogo](../modules/catalog/courses.md), [Aula virtual Zoom](../modules/catalog/zoom-live.md) y [Comunidad](../modules/catalog/community.md) — los módulos que hay detrás de cada herramienta.
