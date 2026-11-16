# Branding whitelabel

Didacta es **whitelabel de serie**: nada de la marca de una instalación concreta vive en el código. Todo lo visual y textual de tu plataforma es configuración de la academia.

## Marca por academia (Community)

En **Administración → Branding** (`/admin/branding`) cada academia configura:

- **Color de marca**: pega el código hex de tu marca (p. ej. `#1E5AA8`) o mueve el matiz y la saturación; los 10 escalones de la paleta se derivan solos.
- **Tipografía**: fuente de titulares y fuente del cuerpo, de una lista cerrada de Google Fonts (por defecto, Sora + Inter).
- **Logo y favicon**: se suben al almacenamiento (logo PNG, JPG, SVG o WebP de hasta 2 MB; favicon de hasta 1 MB, o «usar el logo como favicon») o se pega una URL https. Hay dos presentaciones: **solo el logo** (para logos que ya llevan el nombre) o **logo y nombre**.
- **Pantalla de acceso** (`/signin`): titular (máx. 160 caracteres) y línea de apoyo (máx. 240). Las cifras que aparecen debajo (alumnos activos, cursos publicados) son reales y no se configuran; solo se enseñan cuando la academia tiene al menos 10 alumnos.
- **Restaurar valores por defecto** devuelve el tema al de Didacta.

Los cambios se aplican en vivo en la vista previa y, al guardar, a **todas las personas de la academia**. Este ajuste lo hace el módulo [`mod.theming`](../modules/catalog/theming.md), que también define cómo se presenta la academia en la comunidad (portada, descripción y enlaces) y cuyo detalle está en su página.

!!! note "En modo sencillo"
    En el [modo sencillo](simple-mode.md), **Ajustes → «Logo y color»** reúne lo básico (nombre, logo y color); la pantalla completa sigue abriéndose en su ruta, `/admin/branding`.

## Idioma

Cada academia tiene un **idioma** (español, inglés o portugués de Brasil) y cada persona puede elegir el suyo. Ver [Idiomas](languages.md).

## Página pública de la academia

Lo que ve quien entra **sin sesión** en la raíz del dominio de la academia (`/`):

- **Activada**: una portada pública con el nombre, la portada, la descripción, quién la lleva (nombre y foto de la primera persona que administra, nunca un correo), el **número** de miembros (solo con alumnos suficientes; jamás sus nombres ni caras), el próximo directo (título y fecha, nunca el enlace de la sala) y las **opciones de acceso**: cursos a la venta, membresía y acceso gratuito por solicitud.
- **Desactivada**: directo a la pantalla de acceso (`/signin`).

Se activa y desactiva desde los ajustes de la academia (`PUT /api/v1/admin/tenant-settings/academy`, campo `publicHome`). Mientras nadie lo toque manda el modo: **activada en modo sencillo** y **desactivada en modo completo**, para no cambiar la entrada de las academias que ya existían. Si la academia cobra por [cobros de plataforma](../modules/catalog/payouts.md) y su cuenta aún no puede vender, o tiene las altas congeladas, la portada no ofrece nada a la venta.

**Buscadores**: la academia solo es indexable con la página pública encendida. Entonces la portada, el catálogo (`/catalog`) y cada ficha de venta publicada (`/catalog/<slug>`) salen en `robots.txt` y `sitemap.xml`, con su JSON-LD; apagada, todo queda cerrado (`noindex`). El dominio canónico es el **primario verificado** de la academia, si es uno compartible (no `localhost`, ni una IP, ni `.test`); si no, el de la petición.

## Dominio propio

El tenant se resuelve por el **host** de la petición: cada academia tiene uno o más dominios asociados (el primero se crea en el [asistente de configuración](../installation/setup-wizard.md)).

- Un `super_admin` añade o quita dominios de una academia desde **Administración → Tenants** (`POST`/`DELETE /api/v1/admin/tenants/:id/domains`); **el dominio primario no se puede quitar**.
- **Un host que no pertenece a ninguna academia responde 404**, y el de una academia suspendida muestra una página que lo explica. Es una defensa anti-phishing: con un comodín DNS, cada subdominio libre sería un login funcional. Si tu instalación autoalojada necesita servir la app en cualquier host, `DIDACTA_ALLOW_UNKNOWN_HOSTS=true` restaura el comportamiento anterior.
- La resolución exige **match exacto**: no hay comodines de subdominio. Cada subdominio que quieras usar debe estar dado de alta.
- Poner el dominio en producción (DNS, certificado TLS y enrutado del proxy hacia Didacta) es cosa de tu infraestructura; ver [docker compose](../installation/docker-compose.md).

### Panel «Dominios propios» (Enterprise)

**Administración → Dominios** (`/admin/domains`, capability `feat:custom_domains`) lleva el **registro** de los dominios personalizados de la academia: al añadir uno genera el destino CNAME (el de `DIDACTA_CNAME_TARGET`, por defecto `cname.didacta.io`) y un token de verificación; tú configuras el CNAME y el TXT en tu DNS y pulsas **«Validar»** con el token. Un dominio puede quedar `verified` o `inactive` sin borrarlo.

!!! warning "Lo que el panel no hace todavía"
    En esta versión la verificación es **manual**: compara el token que pegas, **no consulta el DNS**. Y dar de alta un dominio aquí **no lo enruta**: el tráfico solo llega a la academia si el host está registrado en ella (**Tenants**, arriba) y tu proxy y tu certificado lo sirven. Sin licencia, el endpoint responde **402** y la pantalla muestra el aviso de qué desbloquea.

## White-label completo (Enterprise)

Ocultar por completo la marca Didacta («powered by Didacta») y personalizar el **CSS** (máx. 16 KB, sanitizado: se bloquean `@import`, `expression()`, `javascript:` y el cierre de `<style>`) y el **footer HTML** (máx. 4 KB, etiquetas básicas) es la capability Enterprise `feat:white_label`. Sin licencia, esas tarjetas muestran el aviso de qué desbloquean y el backend responde **402** a cualquier intento de guardarlos con contenido — como todas las features Enterprise, [nunca se oculta](../getting-started/editions.md). El resto del branding (logo, colores, tipografía, textos) es Community y no requiere licencia.
