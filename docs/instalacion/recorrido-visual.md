# Recorrido visual: primeros pasos

Esta guía recorre una instalación de Didacta **desde cero**, tal y como la vería cualquier self-hoster nuevo: se construye la imagen del release, se levanta `docker-compose.alpha.yml` sin ningún dato previo, y se completa el camino real de un operador — asistente de configuración, marca, primer curso, alta de alumnado y certificado — con una captura anotada en cada paso.

!!! note "De dónde salen estas capturas"
    Se tomaron contra una instancia aislada, efímera y sin seeds, arrancada desde el tag de un release publicado. Los nombres de organización, curso y personas son de ejemplo (`Academia Demo`, `Admin Demo`, `Alumna Demo`) — sustitúyelos por los tuyos.

## 1 · Primer arranque y asistente de configuración

Al entrar por primera vez, Didacta redirige cualquier ruta a `/setup`. La pantalla de bienvenida resume qué vas a configurar en menos de un minuto.

![Pantalla de bienvenida del asistente de configuración](../assets/recorrido-visual/01-setup-intro.png)

El segundo paso pide el nombre de la organización y el dominio público (el host desde el que accedes, editable).

![Paso de organización: nombre y dominio público](../assets/recorrido-visual/02-setup-organizacion.png)

Después de elegir los módulos activos (los core vienen pre-marcados), el asistente pide los datos del primer administrador — cuenta con rol `super_admin`.

![Paso de cuenta de administrador: nombre, email y contraseña](../assets/recorrido-visual/03-setup-admin.png)

Al terminar, la organización y el administrador ya existen y la sesión queda iniciada: no hace falta volver a entrar.

![Pantalla final del asistente: plataforma creada](../assets/recorrido-visual/04-setup-listo.png)

→ Detalle completo de qué crea el asistente en [Asistente de configuración](setup-wizard.md).

## 2 · Completa tu perfil

La primera vez que cualquier cuenta entra (incluida la del administrador), Didacta pide completar un perfil mínimo — foto obligatoria, nombre, idioma y preferencias de notificación — antes de dejar pasar al resto de la plataforma.

![Onboarding: subir foto de perfil](../assets/recorrido-visual/05-onboarding-foto.png)

## 3 · Inicia sesión

La pantalla de acceso (`/signin`) ya lleva la marca del tenant: nombre, cifras reales (alumnado activo, cursos publicados) y el copy que configures en **Administración → Marca**.

![Pantalla de acceso con la marca del tenant](../assets/recorrido-visual/06-signin.png)

## 4 · Personaliza tu marca

En **Administración → Marca** (`/admin/branding`) subes el logo y eliges el color de marca (`brandHue`). Se aplican al instante a la pantalla de acceso y al catálogo público.

![Admin → Marca: logo subido y color elegido, antes de guardar](../assets/recorrido-visual/07-branding.png)

## 5 · Crea tu primer curso

Desde **Formador → Cursos → Nuevo curso** defines título, descripción y categoría. El *slug* (URL pública) se genera solo desde el título. El curso nace en borrador.

![Alta de un curso nuevo, en borrador](../assets/recorrido-visual/08-curso-nuevo.png)

Dentro del curso añades secciones y, dentro de cada sección, lecciones — aquí una lección de texto.

![Constructor de cursos: alta de una lección de texto](../assets/recorrido-visual/09-curso-leccion-alta.png)

Publicar exige cuatro requisitos (título, descripción, al menos una sección y una lección); el panel te dice en todo momento qué falta.

![Checklist de publicación completo](../assets/recorrido-visual/10-curso-listo-publicar.png)

Con todo listo, el curso pasa a **Publicado**.

![Curso ya publicado](../assets/recorrido-visual/11-curso-publicado.png)

## 6 · Invita a tu alumnado

Sin auto-matriculación pública, la vía normal para que alguien entre a un curso es un **código de invitación**, generado desde el propio curso.

![Código de invitación generado para el curso](../assets/recorrido-visual/12-curso-invitacion-generada.png)

!!! tip "El catálogo público (`/catalogo`) es el de venta"
    `/catalogo` es la ficha pública para cursos **con precio** (mod.billing) — pensada para el visitante que compra con tarjeta sin registrarse antes. Un curso publicado sin ningún precio configurado, como el de este recorrido, no aparece ahí: sigue siendo perfectamente accesible por invitación o por el catálogo interno una vez el alumnado ha entrado. Ver [Tres formas de llenar tu academia](../primeros-pasos/index.md#tres-formas-de-llenar-tu-academia-de-alumnos).

![Catálogo público de venta vacío: este curso no tiene precio configurado](../assets/recorrido-visual/13-catalogo-publico.png)

Para dar de alta a una persona directamente, **Administración → Usuarios → Invitar** crea la cuenta ya activa y le envía un email para que defina su contraseña.

![Alta de una alumna por invitación individual](../assets/recorrido-visual/14-admin-invitar-alumna.png)

## 7 · La alumna entra y se matricula

Desde el enlace del email, la alumna define su contraseña.

![La alumna define su contraseña desde el enlace del email](../assets/recorrido-visual/15-alumna-definir-password.png)

Con sesión ya iniciada, `/cursos` es el catálogo **interno** de la app: lista todos los cursos publicados de la organización, tengan o no precio (a diferencia del `/catalogo` de venta del paso anterior).

![Catálogo interno /cursos, con sesión iniciada](../assets/recorrido-visual/16-alumna-catalogo.png)

En la ficha del curso, canjea el código de invitación que le dio la organización.

![Canje del código de invitación en la ficha del curso](../assets/recorrido-visual/17-alumna-canjear-codigo.png)

Al canjearlo queda matriculada al instante y ve su panel de progreso.

![Alumna matriculada: panel de progreso](../assets/recorrido-visual/18-alumna-matriculada.png)

## 8 · Completa la lección y descarga el certificado

El reproductor de lección muestra el contenido y, para tipos como texto o PDF, un botón para marcarla como completada manualmente.

![Contenido de la lección de texto](../assets/recorrido-visual/19-alumna-leccion.png)

Al alcanzar el umbral de finalización del curso, Didacta emite el certificado de forma automática y el botón del hero cambia a "Descargar certificado".

![Curso completado: certificado listo para descargar](../assets/recorrido-visual/20-alumna-curso-completado.png)

Todos los certificados del alumnado quedan también en `/mis-certificados`, con número correlativo verificable.

![/mis-certificados con el certificado emitido](../assets/recorrido-visual/21-alumna-mis-certificados.png)

## 9 · Un vistazo a Administración → Licencia

Community funciona completo sin licencia. **Administración → Licencia** (`/admin/licencia`) muestra el estado actual y permite activar una licencia Enterprise pegando la clave — la verificación es local, sin llamadas a un servidor externo.

![Admin → Licencia: estado Community, sin licencia Enterprise activa](../assets/recorrido-visual/22-admin-licencia.png)

## Qué no se ve en estas capturas

- El chat flotante (esquina inferior derecha, visible en varias capturas de este recorrido) puede quedar visualmente encima de botones de envío cercanos a esa esquina en pantallas largas — un solape de estilos conocido, sin impacto funcional (el botón sigue siendo clicable).
- El endpoint de matrículas propias (`GET /me/enrollments`) puede tardar hasta ~1 segundo en reflejar una matrícula recién creada. Si acabas de canjear un código y no ves el curso de inmediato en algún listado, recarga.

## Siguiente paso

- [Configuración](../configuracion/index.md) — SMTP real, storage S3, IA.
- [Gestionar módulos](../modulos/gestion.md) — activa o desactiva módulos opcionales.
- [Copias de seguridad](copias-de-seguridad.md) — antes de que haya datos de verdad.
