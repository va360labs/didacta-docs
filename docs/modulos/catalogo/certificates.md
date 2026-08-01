# mod.certificates — Certificados

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Emisión y gestión de certificados PDF al completar un curso: **plantillas personalizables** por organización (logo, copy, color, firmante) con plantilla por defecto, **emisión automática** al recibir `learning.course.completed`, numeración correlativa única por tenant, y **verificación pública** por enlace compartible (la URL que se añade al perfil de LinkedIn).

## Cómo funciona

- Cada certificado emitido guarda un **snapshot inmutable** (alumno, curso, fecha) y el hash SHA-256 del PDF: renombrar el curso después no altera un certificado ya emitido. El PDF se regenera bajo demanda desde el snapshot.
- Doble unicidad: `(tenant, número)` garantiza numeración correlativa y `(tenant, matrícula)` impide emitir dos veces por la misma matrícula.
- La **verificación pública** (`/modules/certificates/verify/:id`) responde `{ number, studentName, courseTitle, issuedAt, valid }` sin exponer nunca email ni datos internos, y contempla la revocación con registro de auditoría.
- El editor de plantillas tiene **preview PDF** con datos de ejemplo, sin persistir nada.

## Dependencias

- Dura: `mod.learning` (la emisión se dispara al completar). Opcional: `mod.courses` (lectura de datos del curso al emitir).

## Modelo de datos

| Tabla | Qué guarda |
| --- | --- |
| `mod_certificates_template` | Plantillas: cuerpo, color, logo, firmante, flag default. |
| `mod_certificates_issued` | Emitidos: número, hash SHA-256, snapshot JSON, clave de storage, revocación. |

## API

Prefijo `/modules/certificates`: los del alumno (`me`, detalle, descarga), plantillas (formador+) y verificación pública. Detalle en [Referencia → Aprendizaje](../../api/referencia/aprendizaje.md#certificados-modulescertificates).

## Eventos

- **Emite**: `certificates.issued`, `certificates.revoked`.
- **Consume**: `learning.course.completed` (auto-emisión).

## Configuración

Sin variables propias; toda la personalización va por plantillas del tenant. Borrar una plantilla en uso o marcada como default responde 409.
