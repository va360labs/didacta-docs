# Límites y diferencias con Community

Cloud y Community son **el mismo producto**: los mismos módulos y la misma imagen publicada. Lo que cambia es **quién opera la instalación** y, con ello, qué puedes configurar y cómo se cobra. Para las tres ediciones, mira [Ediciones](../getting-started/editions.md).

## Lo que cambia

| | Community (self-hosted) | Cloud |
| --- | --- | --- |
| **Quién la opera** | Tú: servidor, base de datos, copias, certificados. | VA360 LABS. |
| **Actualizaciones** | Las aplicas tú ([Actualizar](../installation/upgrading.md)). | Automáticas; no haces nada. |
| **Coste** | Gratis (licencia fair-code); pagas tu infraestructura. | Gratis; Didacta se queda el [5 % de tus ventas](pricing.md). |
| **Cobros** | Con tu propia cuenta de Stripe, sin comisión de Didacta. | Siempre con la cuenta de Stripe de Didacta, que te transfiere tu parte cada semana y se queda el 5 %. No se puede conectar una cuenta de Stripe propia ([Cobros y pagos](payouts.md)). |
| **Países de cobro** | Los que permita tu cuenta de Stripe. | 33 países: la UE, Noruega, Liechtenstein, Reino Unido, Suiza, EE. UU. y Canadá. |
| **Modo del aula** | Completo (menú lateral), con opción de pasar al sencillo. | Sencillo («Empieza aquí»), con opción de pasar al completo. |
| **Dirección** | Tu dominio, como configures. | `<tu-academia>.didacta.io`; sin dominio propio por ahora. |
| **Datos** | Donde los alojes. | Servidores de la UE. |
| **Copias de seguridad** | Las tuyas ([Copias de seguridad](../installation/backups.md)). | Nocturnas y cifradas: 30 días y una al mes durante 12 meses. |
| **Correo** | Tu SMTP. | El servicio de la plataforma con el nombre de tu academia; SMTP propio opcional ([Tu aula](your-academy.md#correo)). |
| **Vídeo** | Tu almacenamiento o enlaces. | Subida a Didacta con [límites](video-and-inactivity.md), o enlaces. |
| **Aulas sin actividad** | No hay política. | Se archivan a los 30 días sin actividad. |
| **Códigos de descuento de Stripe** | Funcionan con tus claves. | Desactivados por ahora. |
| **Soporte** | La comunidad (GitHub). | Por correo, sin plazos garantizados ([Soporte](support.md)). |

## Lo que no cambia

- **Los módulos del producto**: cursos, evaluaciones, certificados, comunidad, mensajería, gamificación, directos, referidos, encuestas… ([catálogo](../modules/catalog/index.md)), salvo los que se indican abajo como no incluidos.
- La **IA con tu propia clave**: si usas el tutor o el generador de contenidos, conectas tu cuenta de OpenAI o Anthropic.
- **El código es público**: lo que ejecuta Cloud es lo que puedes leer y, si quieres, instalar tú.

## Lo que Cloud no incluye

- **Marca blanca**: el pie del aula lleva siempre «Powered by Didacta».
- **SSO (SAML y OIDC) y SCIM**: son capabilities Enterprise y Cloud no las incluye.
- **Módulos de terceros** del marketplace: en Cloud solo funcionan los módulos del producto.
- **Migrador**: no hay ninguna herramienta de migración en Cloud por ahora, ni desde otras plataformas (el módulo Migrador LearnDash no está disponible) ni desde Community ([Baja y migración de datos](data-and-migration.md)).
- **Tu propia cuenta de Stripe**: los cobros van siempre por Didacta.

## Lo que no puedes configurar en Cloud

Cloud no expone la configuración de la instalación, porque no la operas:

- **Variables de entorno** y todo lo que cuelga de ellas (almacenamiento, base de datos, secretos, correo global).
- **Acceso al servidor o a la base de datos.**
- **Dominio propio** y **aplicación móvil**: no incluidos hoy.
- **Acuerdo de nivel de servicio:** Didacta trabaja para que tu aula esté siempre disponible, pero hoy no garantiza un porcentaje de disponibilidad ni tiempos de respuesta, y puede haber interrupciones por mantenimiento (avisa antes siempre que puede).

Sí puedes, en el [modo completo](../installation/visual-walkthrough.md#10-el-modo-sencillo), usar las opciones avanzadas del producto (módulos activables, plantillas de correo, grupos de acceso, etc.).

## Servicios de terceros que conectas tú

Algunas funciones usan servicios que contratas directamente y que no forman parte de Didacta Cloud: **Zoom** para las clases en directo (o pegar el enlace de Zoom, Google Meet o Teams) y **OpenAI o Anthropic** para el asistente de IA. Didacta no responde de ellos.

## Cómo elegir

- Elige **Cloud** si quieres abrir un aula hoy, sin servidor y sin tarjeta, y estás en uno de los [países de cobro](payouts.md#paises-y-monedas).
- Elige **Community** si necesitas tu propio dominio, tu propia infraestructura, cobrar desde un país que Cloud no admite, o integrar con tus sistemas con total control.
- Puedes **cambiar de una a otra**: [Baja y migración de datos](data-and-migration.md).
