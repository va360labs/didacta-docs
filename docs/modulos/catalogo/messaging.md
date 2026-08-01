# mod.messaging — Mensajería

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **community** (desactivable)

## Qué hace

Mensajería nativa con tres tipos de conversación:

- Una **sala de chat por cada espacio** de comunidad.
- **Mensajes directos** entre miembros.
- Un **canal privado automático** de cada alumno con el equipo de profesores (bandeja de consultas del staff).

Con entrega en **tiempo real vía SSE**, presencia, indicador de «escribiendo», marcas de lectura, contador de no-leídos y rate limiting (20 mensajes/min, 30 señales de typing/min).

## Cómo funciona

- La unicidad está garantizada en base de datos: una conversación por espacio, una por alumno en el canal de profesores y una por par de usuarios en DM (clave canónica ordenada), lo que hace el *get-or-create* **idempotente**.
- El nombre del autor se guarda como **snapshot** al enviar (sobrevive a renombres y bajas, y evita N+1).
- Los mensajes borrados conservan el cuerpo para auditoría y se pintan como «Mensaje eliminado».
- El stream SSE se abre con un **ticket efímero** (~60 s) porque `EventSource` no admite headers; eventos: `message.created`, `typing`, `ping`.
- Una cuenta suspendida o borrada no puede usar la mensajería aunque su token siga vivo (verificación contra BD en cada llamada).
- El rate limit usa Redis y degrada a contador local en memoria si Redis falla.

## Dependencias

Opcional: `mod.community` (lectura de espacios para las salas).

## Modelo de datos

`mod_messaging_conversation` (tipada SPACE/DM/FACULTY, con `lastMessageAt` denormalizado) · `mod_messaging_participant` (con `lastReadAt`) · `mod_messaging_message` (snapshot de autor, soporte de adjunto, soft-delete).

## API

Prefijo `/modules/messaging` (+ stream SSE). Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#mensajeria-modulesmessaging).

## Eventos

**Emite**: `messaging.conversation.created`, `messaging.message.sent`. No consume.

## Configuración

Sin configuración por tenant ni variables propias. Requiere Redis para realtime y presencia.
