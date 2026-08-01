# Ediciones: Community, Enterprise y Cloud

Didacta es **un solo producto con tres ediciones**. El código es el mismo; cambia quién lo opera y qué capabilities transversales están activas.

| Edición | Para quién | Qué incluye |
| --- | --- | --- |
| **Community** | Equipos que despliegan y operan ellos mismos. | Todo el código fuente y **todos los módulos**, sin límite de usuarios. Gratuito, self-hosted, bajo la Sustainable Use License. |
| **Enterprise** | Organizaciones con SLA, integraciones a medida y partner certificado. | Community + capabilities transversales del core (SSO SAML/OIDC, SCIM, white-label…), desbloqueadas con una licencia firmada. |
| **Cloud** | Quien quiere arrancar en minutos, sin infraestructura. | Hosting gestionado por VA360 LABS con actualizaciones sin intervención. **En preparación.** |

Precios y contratación: [didacta.io](https://didacta.io).

## El modelo «WordPress matizado»

La línea que separa Community de Enterprise es deliberadamente simple:

- **Los módulos son siempre Community.** Ningún módulo — cursos, certificados, gamificación, Fundae, aula virtual… — se bloquea por licencia, nunca. Lo que instala un self-hoster es el producto completo.
- **Lo único de pago son capabilities transversales del core**: una lista cerrada de funciones de plataforma (federación de identidad, aprovisionamiento, marca blanca…) pensadas para organizaciones grandes. La lista completa está en [Capabilities Enterprise](../enterprise/capabilities.md).

## Cómo se ve el gating en el producto

Didacta sigue la convención de n8n: **nada se oculta**.

- Toda pantalla Enterprise **existe siempre** en el menú y carga con su título y descripción; si no hay licencia, el panel muestra un aviso de upsell en lugar del contenido.
- En la API, los endpoints Enterprise responden **HTTP 402** (`Payment Required`) cuando la capability no está activa.
- Una instalación Community **funciona completa** sin licencia: no hay periodos de prueba ni degradaciones.

## Cómo se activa Enterprise

Con una licencia JWT firmada por VA360 LABS que se introduce en la variable `DIDACTA_LICENSE_KEY` al arrancar. El detalle está en [Licencia Enterprise](../enterprise/licencia.md).
