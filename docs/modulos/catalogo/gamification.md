# mod.gamification — Puntos y retos

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **engagement** (desactivable a propósito)

## Qué hace

Convierte la actividad de la comunidad en un **libro de puntos auditable**, con dos capas deliberadamente distintas:

- **Actividad** — asientos automáticos disparados por eventos del bus (post, comentario, recurso, curso completado, referido…), con pesos bajos y **techo diario** por regla: premian constancia, no volumen.
- **Hitos** — **retos** con prueba obligatoria y revisión humana, con pesos altos.

Encima, los **niveles** se calculan sobre puntos de por vida y nunca bajan; la **clasificación** se calcula por rango de fechas (semana/mes/total) y sí se mueve. Un nivel desbloquea **beneficios** definidos por el operador (sesión 1:1, clase extra…) con cupo por alumno y espera opcional; el alumno los solicita y una persona los atiende.

## Cómo funciona

- La única defensa real contra el doble cobro es la unicidad `(tenant, usuario, sourceKey)`, donde la `sourceKey` identifica el **hecho** (`community.post:<id>`), no el evento — el bus entrega al menos una vez.
- Los **beneficios no tocan los grupos de acceso**: el acceso a cursos es producto de pago y no puede ganarse participando (decisión de producto explícita).
- Moderar contenido (ocultarlo) **revoca** los puntos que dio; restaurarlo los devuelve.
- El **backfill** rellena el ledger con la actividad histórica: idempotente, repetible, excluye contenido oculto y posts por API, y no aplica el techo diario hacia atrás.
- Niveles y retos **nacen vacíos a propósito** (sus nombres y premios son decisiones de marca); solo se siembran las reglas de pesos.

## Dependencias

Opcionales (solo lectura para el backfill): `mod.community`, `mod.learning`, `mod.resources`.

## Modelo de datos

`mod_gamification_ledger_entry` (asiento, append-only salvo revocación) · `_profile` (saldo materializado) · `_rule` · `_level` · `_perk` · `_perk_request` · `_challenge` · `_submission` (una entrega por reto y persona).

## API

Prefijo `/modules/gamification` (miembro + `/admin`). Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#gamificacion-modulesgamification).

## Eventos

- **Emite**: `gamification.points.awarded/revoked`, `gamification.level.changed`, `gamification.challenge.submitted/reviewed`, `gamification.perk.requested/handled`.
- **Consume**: `community.post.created/comment.created/post.hidden/unhidden/comment.hidden/unhidden`, `resources.resource.created/deleted`, `learning.course.completed`, `referrals.referral.attributed`.

## Configuración

Todo por tenant desde Administración → Puntos y retos (pesos, techos, niveles, beneficios, retos). Sin variables de entorno.
