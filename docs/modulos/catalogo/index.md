# Catálogo de módulos

Los 24 módulos incluidos en Didacta Community, **cada uno con su página de detalle**: qué hace, cómo funciona, dependencias, modelo de datos, API, eventos y configuración. Los de categoría **core** están siempre activos; el resto se activa por organización.

## Aprendizaje (core)

| Módulo | Qué hace |
| --- | --- |
| [Cursos y catálogo](courses.md) (`mod.courses`) | Cursos, módulos y lecciones: catálogo y editor del formador, con hook de validación de publicación. |
| [Player, matrícula y progreso](learning.md) (`mod.learning`) | Matriculación, progreso, drip content, invitaciones, SCORM, rutas y competencias. |
| [Quizzes y exámenes](assessments.md) (`mod.assessments`) | Seis tipos de pregunta, autocorrección + corrección manual, intentos y umbrales. |
| [Certificados](certificates.md) (`mod.certificates`) | Plantillas, emisión automática en PDF con snapshot inmutable y verificación pública. |
| [Grupos de acceso](access-groups.md) (`mod.access-groups`) | Entitlement componible: grupos que otorgan cursos, con provenance y refcount. |

## Personas y acceso

| Módulo | Qué hace |
| --- | --- |
| [Inscripción de miembros](member-registration.md) (`mod.member-registration`) *(core)* | Alta con verificadores componibles (Telegram, OTP) y aprobación manual con lookup de pagos. |
| [SSO desde WordPress](wp-sso.md) (`mod.wp-sso`) | Entrada desde una sesión WordPress con token HMAC de un solo uso. |

## Ingresos (core)

| Módulo | Qué hace |
| --- | --- |
| [Stripe Checkout](billing.md) (`mod.billing`) | Pago único de cursos, incluida la compra pública sin cuenta previa. |
| [Suscripciones](subscriptions.md) (`mod.subscriptions`) | Recurrentes con Stripe: gracia, dunning, cancelación y la membresía de `/unete`. |
| [Conexiones de pago](payment-connections.md) (`mod.payment-connections`) | Cuentas Stripe/Woo en solo lectura, tiers, dashboard de suscripciones y espejo de pedidos. |
| [Programa de referidos](referrals.md) (`mod.referrals`) | Comisiones sobre cobros reales con garantía y liquidación manual trazable. |

## Comunidad y engagement

| Módulo | Qué hace |
| --- | --- |
| [Comunidad](community.md) (`mod.community`) *(core)* | Posts, comentarios, reacciones, menciones, espacios, moderación, digest y broadcasts. |
| [Mensajería](messaging.md) (`mod.messaging`) | Salas por espacio, directos y canal de profesores, en tiempo real vía SSE. |
| [Puntos y retos](gamification.md) (`mod.gamification`) | Libro de puntos auditable: reglas con techo diario, niveles, beneficios y retos con prueba. |
| [Biblioteca de recursos](resources.md) (`mod.resources`) | Recursos en colecciones, compartidos por la comunidad, con contador de descargas. |
| [Encuestas y NPS](surveys.md) (`mod.surveys`) | Encuestas anónimas disparadas al terminar cada clase en directo. |

## Inteligencia artificial

Los tres usan el [AI Gateway BYOK](../../configuracion/ia.md) — tú eliges proveedor y clave.

| Módulo | Qué hace |
| --- | --- |
| [Tutor conversacional](ai-tutor.md) (`mod.ai-tutor`) | Tutor por curso con RAG, citas a lecciones y ciclo de calidad con conocimiento validado. |
| [Corrección con rúbrica](ai-grader.md) (`mod.ai-grader`) | Sugerencias de nota por criterio para respuestas abiertas; el formador confirma. |
| [Generador de contenido](ai-content.md) (`mod.ai-content`) | Resúmenes, flashcards y quizzes desde una lección, siempre en borrador. |

## Directo, cumplimiento e integraciones

| Módulo | Qué hace |
| --- | --- |
| [Aula virtual Zoom](zoom-live.md) (`mod.zoom-live`) | Clases en directo con inscripción, calendario, recordatorios y asistencia reconciliada. |
| [Fundae](fundae.md) (`mod.fundae`) | Formación bonificable España: acciones, RLPT, grupos, costes, XML y ZIP de auditoría. |
| [Personalización visual](theming.md) (`mod.theming`) *(core)* | Branding por organización: logo, colores HSL, fuentes y CSS custom sanitizado. |
| [Migrador LearnDash](migrator-learndash.md) (`mod.migrator-learndash`) | Migración completa desde WordPress + LearnDash con ETL, staging y auditoría. |
| [Hello World](hello-world.md) (`mod.hello-world`) | Plantilla de referencia para [crear módulos nuevos](../crear-un-modulo/index.md). |
