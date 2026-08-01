# Catálogo de módulos

Los 24 módulos incluidos en Didacta Community. Los de categoría **Core** están siempre activos (no se pueden desactivar); el resto se activa por organización según necesidad.

## Aprendizaje (core)

| Módulo | Qué hace |
| --- | --- |
| **Cursos y catálogo** (`mod.courses`) | Gestión de cursos, módulos y lecciones: catálogo y editor del formador. |
| **Player + matrícula + progreso** (`mod.learning`) | Matriculación, progreso por lección y reglas de finalización (umbral configurable, 75% por defecto). Incluye itinerarios formativos. |
| **Quizzes y exámenes** (`mod.assessments`) | Quizzes de opción única, múltiple y V/F con autocorrección y umbral de aprobación configurable. |
| **Certificados** (`mod.certificates`) | Plantillas, emisión PDF y descarga de certificados al completar un curso. |
| **Grupos de acceso** (`mod.access-groups`) | Grupos configurables que otorgan acceso a un conjunto de cursos; el acceso se materializa como matrículas con trazabilidad de origen. |

## Personas y acceso

| Módulo | Qué hace |
| --- | --- |
| **Inscripción de miembros** (`mod.member-registration`) *(core)* | Alta de miembros con verificadores componibles (Telegram, OTP por email, registro libre) y validación manual del aprobador. |
| **SSO desde WordPress** (`mod.wp-sso`) | Entrada a Didacta desde una sesión WordPress mediante token corto firmado con HMAC. |

## Ingresos

| Módulo | Qué hace |
| --- | --- |
| **Stripe Checkout** (`mod.billing`) *(core)* | Pago único de cursos: vincula cursos con precios de Stripe y matricula al alumno tras el pago vía webhook idempotente. |
| **Suscripciones** (`mod.subscriptions`) *(core)* | Suscripciones mensuales/anuales con Stripe: activación, periodo de gracia, dunning y cancelación. |
| **Conexiones de pago** (`mod.payment-connections`) *(core)* | Conecta varias cuentas Stripe en solo lectura y reconcilia suscripciones contra los usuarios por email. |
| **Programa de referidos** (`mod.referrals`) *(core)* | Enlace de recomendación por miembro con comisión sobre cobros reales, periodo de garantía y liquidación manual trazable. |

## Comunidad y engagement

| Módulo | Qué hace |
| --- | --- |
| **Comunidad** (`mod.community`) | Posts, comentarios, reacciones y menciones por organización, opcionalmente vinculados a un curso. |
| **Mensajería** (`mod.messaging`) | Salas de chat por espacio, mensajes directos y canal privado alumno↔profesores, en tiempo real vía SSE. |
| **Puntos y retos** (`mod.gamification`) | Convierte la actividad en un libro de puntos auditable: reglas, niveles con beneficios y retos con prueba. |
| **Biblioteca de recursos** (`mod.resources`) | Recursos de la comunidad organizados y buscables: workflows, herramientas, plantillas. |
| **Encuestas y NPS** (`mod.surveys`) | Encuestas anónimas con NPS, disparadas automáticamente al terminar cada clase en directo. |

## Inteligencia artificial

Los tres usan el [AI Gateway BYOK](../configuracion/ia.md) — tú eliges proveedor y clave.

| Módulo | Qué hace |
| --- | --- |
| **Tutor conversacional** (`mod.ai-tutor`) | Tutor IA por curso con RAG sobre el contenido publicado; cita las lecciones relevantes. |
| **Corrección IA con rúbrica** (`mod.ai-grader`) | Corrección de respuestas abiertas con rúbricas del formador; propone nota + feedback y el formador confirma. |
| **Generador de contenido** (`mod.ai-content`) | Borradores de resúmenes, flashcards y quizzes desde una lección, siempre en borrador (human-in-the-loop). |

## Directo, cumplimiento e integraciones

| Módulo | Qué hace |
| --- | --- |
| **Aula virtual Zoom** (`mod.zoom-live`) | Clases en directo: sesiones, inscripción con enlace compartible, acceso a joinUrl/grabación por matrícula y asistencia reconciliada. |
| **Fundae** (`mod.fundae`) | Cumplimiento regulatorio España: acciones formativas (código, modalidad, horas, fechas), grupos y participantes, export XML y ZIP de auditoría. |
| **Personalización visual** (`mod.theming`) | Branding por organización: logo, favicon, color primario, fuentes y CSS custom sanitizado. |
| **Migrador LearnDash** (`mod.migrator-learndash`) | Importa cursos, usuarios, matrículas, media y progreso desde WordPress + LearnDash con ETL, staging e idempotencia. |
| **Hello World** (`mod.hello-world`) | Módulo de ejemplo: plantilla de referencia para [crear módulos nuevos](crear-un-modulo/index.md). |
