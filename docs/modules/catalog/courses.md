# mod.courses — Cursos y catálogo

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Es el corazón del catálogo: cursos con `slug`, título y estado (`DRAFT` → `PUBLISHED` → `ARCHIVED`), **módulos** como agrupación ordenada dentro del curso y **lecciones** tipadas (`VIDEO`, `HTML`, `PDF`, `TEXT`, `QUIZ`, `SCORM`) con contenido JSON flexible. Incluye categorías curadas por el administrador (nombre, color, icono) y operaciones de edición avanzadas: reordenar módulos y lecciones en bloque (drag & drop) y mover lecciones entre módulos.

## Cómo funciona

- La **publicación pasa por un hook abierto**: antes de publicar, `mod.courses` ejecuta `courses.publish.validate` y cualquier módulo registrado puede añadir razones de bloqueo (por ejemplo, `mod.fundae` exige objetivos y duración). Si hay razones, la publicación falla con 422 y la lista de motivos.
- Los borrados son **soft-delete** con cascada lógica: eliminar un módulo o lección preserva el progreso histórico de los alumnos.
- Es la pieza de la que cuelgan casi todos los demás módulos (learning, access-groups, billing, assessments, ai-content, ai-tutor…).

## Configuración

- **Activación**: módulo de categoría `core`, siempre activo en todos los tenants. No aparece en la pestaña «Módulos» de `/admin/settings?tab=modules` (esa pestaña solo lista los módulos desactivables) y la API rechaza desactivarlo; si en base de datos queda una fila con `enabled=false`, el arranque del servidor la re-habilita solo.
- **Ajustes por tenant**: no expone ninguno. Toda la configuración es por curso, desde el builder del formador.
- **Categorías curadas**: el administrador las gestiona en `/admin/courses/categories` (pantalla «Categorías de cursos»): nombre, «Color» e «Icono». El nombre casa por texto exacto con la categoría escrita en el curso; el catálogo y la card del curso pintan el chip con ese estilo.
- **Variables de entorno**: ninguna es del módulo en sí. Relacionadas con el contenido: `LESSON_IFRAME_ALLOWED_HOSTS` (hosts extra de `<iframe>` en lecciones HTML) y `BUNNY_STREAM_*` (subida de vídeo, solo Didacta Cloud; ver «Vídeo de las lecciones»). Las imágenes de portada y los paquetes SCORM se guardan en el backend de archivos de la instancia (`/admin/settings?tab=storage`, disco local o S3-compatible).
- **Licencia**: todo el módulo es Community; ninguna función exige capabilities de Enterprise.

!!! note "Capturas en modo completo"
    Las capturas de esta página son del **modo completo** (menú lateral), el de una instalación nueva. En **modo sencillo** las mismas rutas siguen disponibles desde «Más», salvo las pantallas que ese modo sustituye por una versión simplificada (el editor de cursos, Ajustes, Ventas, Alumnos, Directos, la comunidad y Referidos); para usarlas tal como se ven aquí, cambia al modo completo en **Ajustes → «Usar el modo completo»**.

## Uso paso a paso

### Crear y montar un curso (formador)

1. En `/studio/courses` («Mis cursos»), pulsa «Crear curso nuevo». Rellena «Título» y «Slug» (se genera solo desde el título, en kebab-case: será la URL pública del curso) y, si quieres, descripción, «Categoría» y «Duración estimada (min)». Pulsa «Crear curso»: el curso nace en `DRAFT`.

    ![Listado «Mis cursos» del formador con cursos en borrador, publicados y archivados](../../assets/modules/courses/courses-1-my-courses.png)

2. En el builder (`/studio/courses/<id>`), pulsa «Añadir sección», escribe el título y confirma con «Crear sección». Las secciones agrupan lecciones relacionadas.
3. Dentro de cada sección, «Añadir lección»: elige el «Tipo» (Vídeo, HTML, PDF, Texto, Quiz o SCORM), escribe el título y pulsa «Crear».
4. Pulsa «Editar» sobre la lección para cargar su contenido según el tipo: «URL del vídeo» (YouTube, Bunny Stream o archivo `.mp4`/`.webm`/`.m3u8`) con «Recursos y capítulos» clicables (y, opcionalmente, «Transcripción (para el tutor IA)»: pega el texto o sube un `.srt`, `.vtt` o `.txt`; el alumno no la ve, la lee el [tutor IA](ai-tutor.md)), «URL del PDF», «HTML», «Texto», quiz vinculado (ver [mod.assessments](assessments.md)) o «Subir paquete SCORM» (un `.zip` con `imsmanifest.xml` en la raíz). El campo «Fecha de publicación (opcional)» programa la lección: con fecha futura aparece en el listado pero bloqueada hasta ese momento.

    ![Builder del curso con secciones, lecciones y la tarjeta de publicación](../../assets/modules/courses/courses-2-builder.png)

5. Con «Editar» en la cabecera ajustas los metadatos: descripción, «Categoría», «Duración estimada (min)», «Imagen destacada» (botón «Optimizar» la recomprime a WebP), «Vídeo destacado (URL)» y «URL de compra externa» si la venta de este curso ocurre en una página externa.
6. Reordena secciones y lecciones arrastrando (drag & drop); una lección también puede moverse a otra sección. Los borrados son soft-delete: dejan de mostrarse pero el progreso histórico de los alumnos se conserva.
7. La tarjeta «Antes de publicar» lista los requisitos: «Título del curso», «Descripción», «Al menos una sección» y «Al menos una lección». Con todo cumplido cambia a «Listo para publicar». Pulsa «Publicar»; si un módulo enganchado al hook bloquea la publicación (p. ej. FUNDAE), la API responde 422 con los motivos.
8. «Archivar» retira el curso del catálogo sin borrar nada; «Desarchivar» lo devuelve a `DRAFT` (no vuelve a publicarse solo: lo revisas y pulsas «Publicar» otra vez).

### Vídeo de las lecciones: subida y protección

- **Subir el vídeo desde el editor (solo Didacta Cloud).** En una lección de vídeo, además de pegar un enlace, el editor ofrece «Sube el vídeo de esta lección»: arrastra el archivo o elígelo, con «Pausar» y «Reanudar» si se corta la conexión. Tras la subida el vídeo se prepara en unos minutos (puedes seguir editando) y hay que pulsar «Guardar cambios» para que lo vean los alumnos. Solo existe si la instancia tiene `BUNNY_STREAM_API_KEY`; sin ella (el caso de una instalación propia) el editor solo admite enlaces y las rutas `/video/uploads` responden 404. Límites por vídeo: 4 GB y 2 horas. El espacio total depende de si ya has vendido: antes de la primera venta, 5 h almacenadas, hasta 720p y 100 h vistas al mes; después, 50 h (más 10 h por cada 30 € de comisión del operador en los últimos 12 meses), hasta 1080p y sin tope de horas vistas. La tarjeta «Tu espacio de vídeo» del editor muestra lo usado.
- **Protección de vídeo** (`/admin/security`, tarjeta «Protección de vídeo», solo `super_admin` y `tenant_admin`). Para los vídeos de Bunny Stream que enlazas a mano: con la protección encendida, cada reproducción comprueba que el alumno recibe el contenido de esa lección y le da un enlace firmado que caduca a los 30 minutos. Para encenderla guardas la «Token authentication key» de cada librería (se almacena cifrada y no se vuelve a mostrar), activas el token en Bunny y pulsas «Comprobar». Ajustes: «Protección activada» (por defecto sí), «Marca de agua con el email del alumno» (por defecto sí), «Vídeos distintos por hora» (30) y «Vídeos distintos por día» (100) por cuenta; formadores y administradores no tienen tope. Los vídeos subidos desde el editor ya van firmados por la plataforma y no necesitan clave.
- **HTML saneado.** El HTML de las lecciones y la descripción del curso se sanean **en el servidor al guardar**: se descartan `<script>`, `<style>` y los enlaces `javascript:`. En lecciones HTML solo se admiten `<iframe>` de YouTube, Vimeo y Bunny Stream (más los hosts que el operador añada en `LESSON_IFRAME_ALLOWED_HOSTS`, separados por comas).
- **Quién recibe el contenido.** `GET /modules/courses/:id` devuelve a un alumno el temario completo, pero el contenido de cada lección solo si tiene matrícula viva y la lección está liberada (drip, fecha de publicación, periodo de prueba o nivel); formadores y administradores lo ven siempre completo.

### Curar las categorías del catálogo (admin)

1. En `/admin/courses/categories`, crea cada categoría con «Nombre», «Color» e «Icono» y pulsa «Crear categoría». La vista previa muestra el chip resultante.
2. El builder ofrece este listado al formador como desplegable de «Categoría». Si se elimina una categoría, los cursos que la tenían siguen mostrando el nombre como texto plano.

    ![Pantalla de categorías curadas con color e icono](../../assets/modules/courses/courses-3-categories.png)

### Lo que ve el alumno

- El catálogo (`/courses`) lista solo los cursos `PUBLISHED`, con búsqueda y filtros por temática e idioma.
- La ficha (`/courses/<slug>`) muestra el hero (vídeo destacado > imagen > fallback), la descripción y el temario. Un formador o admin puede abrir la ficha de un curso sin publicar en modo «Vista previa», sin matricularse.

    ![Catálogo de cursos del alumno con chips de categoría](../../assets/modules/courses/courses-4-catalog.png)

## Dependencias

- Duras: ninguna.
- Opcional: `mod.certificates` (valida la plantilla de certificado asignada al curso).

## Modelo de datos

| Tabla | Qué guarda |
| --- | --- |
| `mod_courses_course` | Curso: slug, título, estado, categoría, plantilla de certificado. |
| `mod_courses_module` | Agrupación ordenada dentro del curso. |
| `mod_courses_lesson` | Lección: tipo, contenido JSONB, duración, fecha de publicación programada. |
| `mod_courses_category` | Categorías curadas (nombre, color, icono). |

## API

Prefijo `/modules/courses`: CRUD de cursos/módulos/lecciones, categorías gestionadas, transiciones de estado y bloque de ordenación. Detalle en [Referencia → Aprendizaje](../../api/reference/learning.md#cursos-modulescourses).

## Eventos y hooks

- **Emite**: `courses.course.created/updated/published/archived/unarchived`, `courses.module.created/deleted`, `courses.lesson.created/updated/moved/moved-to-module/deleted`, `courses.lessons.reordered` y `courses.modules.reordered`.
- **Expone el hook** `courses.publish.validate` (el único hook activo del producto — el patrón de referencia).
