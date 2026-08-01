# mod.community — Comunidad

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

El feed social del tenant: **posts** (opcionalmente ligados a un curso y a un espacio), **comentarios** con un nivel de respuesta, **reacciones** por emoji, **menciones** `@usuario` con autocomplete, **tags curados** por el admin (nombre, color, icono) y **espacios** administrables (los 4 de sistema son editables pero no eliminables). Encima: moderación, pinning de posts, directorio de miembros, digest semanal por email y **avisos masivos** (broadcasts) con baja por enlace.

## Cómo funciona

- Distinción deliberada entre **ocultado de moderación** (`hidden`, reversible, la fila se preserva con motivo) y **borrado del autor** (soft-delete).
- Un post con `notifyAll` (solo admin) genera además un broadcast por email y campana a todos los miembros; `important` ignora el opt-out del receptor.
- Los broadcasts se envían **por lotes con throttling** (para cuidar la reputación SMTP) y llevan cursor de reanudación para reintentos.
- Publicación por **API externa** (`/community-api`, con API key de scope `community:post` cuyo dueño sea admin): los posts entran con `source='api'` — así se distingue el contenido automatizado.
- El digest semanal se programa con `COMMUNITY_DIGEST_CRON` (lunes 09:00 UTC por defecto) y respeta el opt-out por usuario.

## Dependencias

Opcional: `mod.courses` (posts ligados a curso).

## Modelo de datos

`mod_community_post` · `_comment` · `_reaction` · `_mention` · `_tag` · `_space` · `_broadcast` (con estado y cursor) · `_user_pref` (preferencias, hoy el opt-out del digest).

## API

Prefijo `/modules/community` + `/community-api` (integraciones) + baja pública por token. Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#comunidad-modulescommunity).

## Eventos

**Emite**: `community.post.created`, `community.comment.created`, `community.reaction.added` (+ menciones). No consume. Sus eventos alimentan la [gamificación](gamification.md) y los [webhooks salientes](../../api/convenciones.md#webhooks-salientes).

## Configuración

Espacios, tags y digest se configuran desde el panel. Workers: `COMMUNITY_DIGEST_CRON`, `COMMUNITY_BROADCAST_BATCH_SIZE` (5), `COMMUNITY_BROADCAST_INTERVAL_MS` (10 s).
