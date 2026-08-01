# mod.fundae — Cumplimiento Fundae

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **compliance** (desactivable)

## Qué hace

Cumplimiento regulatorio para **formación bonificable en España** (RD 694/2017):

- **Acciones formativas** con código, modalidad (`PRESENCIAL`/`TELEFORMACION`/`MIXTA`), horas, fechas y bloques de contenido.
- **Empresas bonificadas** con NIF validado (checksum DNI/NIE/CIF), CCC y crédito anual.
- **Notificaciones RLPT** a la representación legal de las personas trabajadoras, con el plazo legal de 15 días naturales calculado automáticamente y la evidencia (PDF) en el Evidence Vault con hash.
- **Grupos bonificables** con ciclo de vida (`DRAFT → ACTIVE → CLOSED/CANCELLED`), costes imputados y matrícula nominal de participantes.
- **Finalización** con umbral configurable (75% por defecto): APTO / NO_APTO / EN_CURSO por participante, con modo `preview`.
- **Exports**: XML de inicio y de fin de grupo, y un **ZIP de auditoría** completo (manifest SHA-256 + XMLs + CSVs + evidencias RLPT) verificable offline con una herramienta sin dependencias.

## Cómo funciona

- Arrancar un grupo valida la **antelación RLPT** (15 días) y el **crédito disponible** de la empresa; cerrar el grupo debita el crédito consumido.
- El NIF de una empresa es **inmutable** tras el alta (cambio de NIF = empresa nueva, por trazabilidad).
- La suma de horas de los bloques no puede superar las horas de la acción (Fundae lo verifica al subir el XML).
- Es el consumidor natural del hook `courses.publish.validate`: puede bloquear la publicación de un curso si faltan objetivos o duración.
- Hay vistas diferenciadas por rol: admin, **auditor** (solo lectura, redactada) y `empresa_manager` (solo sus empleados). El rol formador no accede a Fundae.

## Dependencias

Opcionales: `mod.courses` (validación de curso vinculado) y `mod.learning` (matrículas y completitud).

## Modelo de datos

`mod_fundae_action` · `_block` · `_company` · `_rlpt_notice` · `_group` · `_cost` · `_group_participant`.

## API

Prefijo `/modules/fundae` (acciones, exports) + superficie admin en `/admin/fundae/*` (empresas, RLPT, grupos, costes, participantes). Detalle en [Referencia → Aprendizaje](../../api/referencia/aprendizaje.md#fundae-modulesfundae-todo-admin-el-rol-formador-no-accede) y [Referencia → Administración](../../api/referencia/administracion.md#fundae-admin-el-rol-formador-no-accede).

## Eventos

**Emite** 24 eventos que cubren todo el ciclo: `fundae.action.*`, `fundae.company.*`, `fundae.rlpt.notice.*`, `fundae.group.*` (incluidos los de generación de XML y ZIP), `fundae.cost.*`, `fundae.export.generated`. No consume.

## Configuración

Umbral de finalización configurable por grupo (75% por defecto). Sin variables de entorno propias.
