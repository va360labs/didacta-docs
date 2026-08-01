# mod.surveys — Encuestas y NPS

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **engagement** (desactivable)

## Qué hace

Encuestas **anónimas** de la comunidad. El caso principal: al terminar cada clase en directo (`zoom.session.ended`) se crea automáticamente la encuesta post-clase (NPS + valoración), se avisa a los inscritos y el admin ve los **resultados agregados** (NPS, medias, respuestas de texto). Incluye un worker de recordatorio a quienes no han respondido tras 24 h.

## Cómo funciona

- El anonimato es **estructural**: la respuesta no guarda `userId`, solo un `respondentHash` = HMAC(encuesta:usuario) con secreto del servidor — permite deduplicar (una respuesta por persona) sin poder identificar al autor.
- Una única encuesta por sesión de Zoom y un único recordatorio por encuesta (garantizado con uniques en BD).
- El admin puede crear la encuesta de una clase sin esperar al webhook, cerrar encuestas (dejan de aceptar respuestas con 409) y forzar el barrido de recordatorios.
- El modelo contempla también encuestas al completar curso y generales del tenant (`POST_COURSE`, `GENERAL`), aún sin disparador automático.

## Dependencias

Opcional: `mod.zoom-live` (lectura de inscritos para avisar al crear la encuesta post-clase).

## Modelo de datos

`mod_surveys_survey` (tipo, estado OPEN/CLOSED, marca de recordatorio) · `_question` (tipada y posicionada) · `_response` (anónima, solo `respondentHash`) · `_answer` (valor numérico o texto).

## API

Prefijo `/modules/surveys` (alumno + admin). Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#encuestas-modulessurveys-respuestas-anonimas).

## Eventos

- **Emite**: `surveys.survey.created`, `surveys.response.submitted`.
- **Consume**: `zoom.session.ended` (crea la encuesta post-clase).

## Configuración

Sin ajustes por tenant. Workers: `SURVEYS_REMINDER_CRON` (cada 15 min), `SURVEYS_REMINDER_DELAY_HOURS` (24), `SURVEYS_HASH_SECRET` (cae a `AUTH_SECRET`).
