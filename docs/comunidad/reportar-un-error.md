# Reportar un error

Didacta está en **alpha**: los reportes de quienes lo instalan son la principal fuente de arreglos. Esta página explica qué mirar antes de abrir una issue, qué datos hacen falta y cómo enviarlos sin filtrar información sensible.

Los errores se reportan como **issue en GitHub** con plantilla:

[Abrir una issue :material-github:](https://github.com/va360labs/didacta-io/issues/new/choose){ .md-button .md-button--primary }

!!! danger "¿Es un problema de seguridad?"

    **No abras una issue pública.** Las vulnerabilidades (bypass de autenticación o de RLS, fuga de datos entre tenants, RCE, inyección SQL, escalada de privilegios…) se reportan por email a **`security@didacta.io`** siguiendo la [política de seguridad](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md). Publicarlas en abierto deja expuestas a todas las instalaciones self-host hasta que haya parche.

## Antes de abrir la issue

1. **Mira [Solución de problemas](../instalacion/solucion-de-problemas.md).** Los fallos más frecuentes de instalación (pgvector ausente, migración a medias, puerto ocupado, `AUTH_SECRET` corto, correos que se quedan en Mailpit) están ahí con su arreglo.
2. **Actualiza a la última alpha.** Durante la alpha cerrada solo se da soporte a la última versión publicada; si el bug es de una alpha anterior, se te pedirá reproducirlo en la actual. Ver [Actualizar](../instalacion/actualizacion.md).
3. **Busca en las [issues abiertas](https://github.com/va360labs/didacta-io/issues).** Si ya está reportado, añade tu caso como comentario (tu versión, tu sistema operativo, tus logs) en vez de abrir una duplicada: sirve más para reproducirlo.
4. **Comprueba que es un error y no una duda de uso.** Las preguntas de uso, instalación o arquitectura van a [GitHub Discussions](https://github.com/va360labs/didacta-io/discussions), que es donde se responden.

## Qué plantilla usar

| Situación | Dónde va |
| --- | --- |
| Algo no funciona: error, pantalla en blanco, 500, dato incorrecto | Plantilla **🐛 Bug report** |
| Funciona, pero es confuso, lento o está mal organizado | Plantilla **💬 Feedback general** |
| Falta una funcionalidad o quieres un cambio de comportamiento | Plantilla **✨ Feature request** |
| Duda de uso, instalación o configuración | [Discussions](https://github.com/va360labs/didacta-io/discussions) |
| Vulnerabilidad de seguridad | `security@didacta.io` (**nunca** issue) |
| Uso comercial, Enterprise o Cloud | `licensing@didacta.io` |

## Datos que necesitamos

La plantilla de bug los pide uno a uno. Estos son los que más cuesta reunir después:

### Versión exacta

Cualquiera de estas tres vale:

```bash
# 1. La versión que responde la API (no necesita autenticación)
curl -fsS http://localhost:4000/healthz
# → {"status":"ok","service":"api","version":"0.0.1-alpha.101",...}

# 2. El tag de imagen que tienes desplegado
docker compose -f docker-compose.alpha.yml images didacta

# 3. Lo que hayas fijado en .env
grep DIDACTA_IMAGE_TAG .env
```

### Cómo lo estás ejecutando

`docker compose -f docker-compose.alpha.yml`, `pnpm dev` en local, u otra cosa (Kubernetes, Postgres gestionado, proxy inverso delante…). Si tu montaje se sale del [Docker Compose de referencia](../instalacion/docker-compose.md), dilo: cambia mucho el diagnóstico.

### Pasos para reproducir

Numerados, desde una instalación recién arrancada si puedes, e indicando **con qué rol** los haces (administrador, formador, alumno). Un bug que solo aparece con un rol concreto y no se dice, se cierra como no reproducible.

### Logs

```bash
# Los últimos 200 registros de la aplicación, a un fichero que puedas adjuntar
docker compose -f docker-compose.alpha.yml logs --no-color --tail 200 didacta > didacta-logs.txt

# Estado de los servicios y sus healthchecks
docker compose -f docker-compose.alpha.yml ps
```

### Si el error es de interfaz

Abre las herramientas de desarrollo del navegador (`F12`) y añade:

- La **consola**: el error en rojo, completo.
- La pestaña **Red**: la petición que falla — URL, método y código de respuesta (401, 402, 403, 500…). Un **402** no es un fallo: es una capability [Enterprise](../enterprise/index.md) que tu licencia no cubre.
- Una **captura** de la pantalla, que suele ahorrar tres mensajes de ida y vuelta.

## Anonimiza antes de pegar

Los logs y las capturas de una instalación real llevan datos de personas y credenciales. Repásalos antes de subirlos — una issue de GitHub es pública e indexable, y borrarla después no deshace el que alguien ya la haya leído.

| Sustituye | Por |
| --- | --- |
| Emails y nombres de alumnos, formadores o clientes | `alumna@example.com`, `Nombre Apellido` |
| Tokens JWT, cookies de sesión, claves de API | `<REDACTED>` |
| `AUTH_SECRET`, contraseñas de base de datos, credenciales SMTP | `<REDACTED>` |
| Claves de Stripe (`sk_live_…`, `whsec_…`) y de proveedores de IA | `<REDACTED>` |
| Tu clave de licencia Enterprise (JWT firmado) | `<REDACTED>` |
| Dominios internos o IPs privadas, si te importan | `ejemplo.com`, `10.0.0.x` |

!!! warning "Si ya has publicado una credencial"

    Dala por comprometida: rótala en tu instalación (regenera la clave, cambia la contraseña) además de editar la issue. Cambiar `AUTH_SECRET` invalida todas las sesiones abiertas, lo cual es lo que quieres en ese caso.

## Cómo escribir el reporte

- **Un error por issue.** Tres bugs en una sola issue se arreglan a distinto ritmo y acaban bloqueándose entre sí.
- **Título concreto.** «El certificado sale sin el nombre del alumno cuando el curso tiene dos módulos» es útil; «No funciona» no lo es.
- **Separa lo que esperabas de lo que pasó.** Es la parte que decide si el comportamiento es un fallo o una decisión de producto.
- **Di si es reproducible siempre o intermitente**, y cuántas veces de cuántas. Los fallos de carrera se buscan de forma muy distinta.
- **Menciona lo que ya has descartado.** Si reiniciaste, si probaste en otro navegador, si pasa también en una instalación limpia.

??? example "Ejemplo de reporte completo"

    **Título:** El canje de código de invitación devuelve 500 si el curso ya está archivado

    **Versión:** 0.0.1-alpha.101 · **Ejecución:** `docker compose -f docker-compose.alpha.yml` · **SO:** Ubuntu 22.04

    **Pasos:**

    1. Como administradora, creo un curso y genero un código de invitación.
    2. Archivo el curso desde el listado de administración.
    3. Como alumna, entro en `/canjear` e introduzco el código.

    **Esperado:** un mensaje diciendo que el curso ya no está disponible.

    **Real:** error 500. En los logs: `TypeError: Cannot read properties of null (reading 'id')`.

    **Reproducible:** siempre, 5 de 5, también en instalación limpia.

    Adjunto `didacta-logs.txt` con los emails sustituidos por `@example.com`.

## Qué pasa después

1. Se revisa el reporte y se etiqueta. Si falta algún dato para reproducirlo, se pide en un comentario — una issue sin respuesta a esa petición acaba cerrándose por falta de información, y siempre se puede reabrir.
2. Si se reproduce, se confirma en la issue y se prioriza según el impacto: pérdida de datos y fallos que bloquean la instalación van primero.
3. Al mergearse el arreglo, la issue se cierra referenciando el commit, y el cambio sale en la siguiente versión con entrada en el [CHANGELOG](https://github.com/va360labs/didacta-io/blob/main/CHANGELOG.md).

No hay compromiso de plazo para bugs durante la alpha; los plazos comprometidos son los de la [política de seguridad](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md), que solo aplican a vulnerabilidades.

## ¿Y si quieres arreglarlo tú?

Los parches de la comunidad son bienvenidos. Abre igualmente la issue —sirve de referencia para el PR— y sigue la guía de [Contribuir](contribuir.md): requiere firmar el CLA, Conventional Commits y tests para lógica de negocio.
