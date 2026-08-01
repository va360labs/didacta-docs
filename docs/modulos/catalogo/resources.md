# mod.resources — Biblioteca de recursos

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **engagement** (desactivable)

## Qué hace

Biblioteca de recursos de la comunidad, organizados en **colecciones** y buscables: workflows de las clases, skills, directorio de herramientas de IA y plantillas. Cada recurso es un archivo subido al storage del tenant (`FILE`) o un enlace externo (`LINK`), puede quedar vinculado a la clase en directo que lo generó y lleva contador de descargas.

## Cómo funciona

- La primera visita siembra **6 colecciones por defecto**; el staff puede crear más, con portada.
- Cualquier miembro puede **compartir recursos**; borrar puede el autor (lo suyo) o el staff (cualquiera).
- El borrado de una colección con recursos dentro está **bloqueado a propósito** (`Restrict`, no cascade): hay que vaciarla antes — nunca se pierde contenido en silencio.
- `POST /:id/download` registra la descarga/apertura y devuelve la URL: el contador alimenta la ordenación por popularidad.

## Dependencias

Ninguna. La referencia a la clase (`zoomSessionId`) es un ID lógico sin FK.

## Modelo de datos

`mod_resources_collection` (título único por tenant, orden, flag default) · `mod_resources_resource` (kind FILE/LINK, URL, autor, clase de origen, descargas).

## API

Prefijo `/modules/resources`. Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#recursos-modulesresources).

## Eventos

**Emite**: `resources.resource.created`, `resources.resource.deleted`, `resources.collection.created` (los dos primeros puntúan en [gamificación](gamification.md)). No consume.

## Configuración

Sin configuración: solo el toggle de activación del módulo.
