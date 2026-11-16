# Idiomas

La interfaz de Didacta, los correos y los textos que el servidor escribe por ti existen en **tres idiomas**: **español**, **inglés** y **portugués de Brasil**. Los contenidos que escribes tú (cursos, lecciones, posts…) son siempre en el idioma en el que los escribes: no se traducen solos.

## Dos niveles

| | Qué es | Dónde se elige |
| --- | --- | --- |
| **Idioma de la academia** | El de la página de venta y el catálogo, el de los correos a quien aún no ha elegido idioma, el de los textos que el servidor escribe para todos (anuncio de un directo en la comunidad, espacios iniciales de la comunidad…) y lo que ve quien entra sin cookie ni perfil. | En el [asistente de configuración](../installation/setup-wizard.md) (propone el idioma de tu navegador si lo hay), en el **alta en 3 preguntas** del [modo sencillo](simple-mode.md) y en **Ajustes → General** (`PUT /api/v1/admin/tenant-settings/academy`, campo `locale`: `es`, `en` o `pt`). Lo cambia un `tenant_admin` o `super_admin`. |
| **Idioma de cada persona** | El de su interfaz y el de sus correos y avisos. | En su perfil (`/account`) o en el selector de idioma. Es una preferencia personal. |

### Qué idioma se ve

La interfaz se pinta con el primero de estos que exista: la **cookie `didacta_locale`** (lo que eligió el visitante en el selector) → el **idioma del perfil** de la persona → el **idioma de la academia** → español. En las páginas públicas (como la compra sin cuenta) manda la cookie. Los correos van en el idioma del **perfil de quien los recibe**; si todavía no lo tiene, en el de la academia.

!!! info "Variantes"
    El perfil guarda `es-ES`, `es-AR`, `en-US` o `pt-BR`. `es-AR` usa el mismo catálogo en español y solo cambia el formato de fechas y números. Cualquier otro portugués (por ejemplo `pt-PT`) se sirve como `pt-BR`, y cualquier otro inglés como `en-US`.

## Qué está traducido

- **Interfaz** completa de la web, con una guarda en CI (`messages-parity`) que impide que un idioma entre a medias.
- **Correos y avisos**: las plantillas del hub de notificaciones (bienvenidas de compra, recordatorios, digest, suscripciones…) están en los tres idiomas; cada plantilla se puede **personalizar por academia** en **Administración → Emails** (ver [Correo](email.md)). El comprador de un checkout anónimo recibe sus correos en el idioma con el que estaba navegando.
- **Textos de cumplimiento**: la renuncia al derecho de desistimiento, el certificado en PDF (es, en, pt) y los `.ics` de los directos.
- **Asistente del modo sencillo y mensajes automáticos** de [mod.retention](../modules/catalog/retention.md): plantillas escritas en el idioma de la academia, con variables válidas en los tres idiomas.

!!! note "Qué no entra"
    Los textos escritos por los equipos (cursos, lecciones, posts, plantillas editadas a mano) se quedan como se escribieron. El panel de administración no ofrece idiomas fuera de estos tres; un idioma nuevo exige un catálogo completo de mensajes en el código.
