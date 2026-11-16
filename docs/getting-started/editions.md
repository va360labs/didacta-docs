# Ediciones: Community, Enterprise y Cloud

Didacta es **un solo producto con tres ediciones**. El código es el mismo; cambia quién lo opera y qué capabilities transversales están activas.

| Edición | Para quién | Qué incluye |
| --- | --- | --- |
| **Community** | Equipos que despliegan y operan ellos mismos. | Todo el código fuente y **todos los módulos**, sin límite de usuarios. Gratuito, self-hosted, bajo la Sustainable Use License. |
| **Enterprise** | Organizaciones con SLA, integraciones a medida y partner certificado. | Community + capabilities transversales del core (SSO SAML/OIDC, SCIM, white-label…), desbloqueadas con una licencia firmada. |
| **Cloud** | Quien quiere abrir una academia en minutos, sin infraestructura. | El mismo producto, alojado y mantenido por VA360 LABS, con actualizaciones sin intervención. Gratis: Didacta se queda un 5 % de cada venta que haces. Ver [Didacta Cloud](../cloud/index.md). |

Precios y contratación: [didacta.io](https://didacta.io). Cloud no necesita contratación: te das de alta en [didacta.io/es/start](https://didacta.io/es/start) y [esta sección](../cloud/index.md) explica cómo funciona.

## Dónde está la línea

La línea que separa Community de Enterprise es deliberadamente simple:

- **Los módulos son siempre Community.** Ningún módulo — cursos, certificados, gamificación, Fundae, aula virtual… — se bloquea por licencia, nunca. Lo que instala un self-hoster es el producto completo.
- **Lo único de pago son capabilities transversales del core**: una lista cerrada de funciones de plataforma (federación de identidad, aprovisionamiento, marca blanca…) pensadas para organizaciones grandes. La lista completa está en [Capabilities Enterprise](../enterprise/capabilities.md).

## Dónde encaja Cloud

Cloud no es una línea de producto distinta: ejecuta **la misma imagen publicada** que se instala en Community, con los mismos módulos. Cambia quién la opera —VA360 LABS— y cómo se cobra: los pagos de tus alumnos los recibe Didacta y te transfiere tu parte, quedándose un 5 % de cada venta. Qué se puede y qué no configurar, y cómo pasar de una edición a otra, está en [Límites y diferencias con Community](../cloud/cloud-vs-community.md) y en [Baja y migración de datos](../cloud/data-and-migration.md).

## Cómo se ve el gating en el producto

En Didacta **nada se oculta**.

- Toda pantalla Enterprise **existe siempre** en el menú y carga con su título y descripción; si no hay licencia, el panel muestra un aviso de upsell en lugar del contenido.
- En la API, los endpoints Enterprise responden **HTTP 402** (`Payment Required`) cuando la capability no está activa.
- Una instalación Community **funciona completa** sin licencia: no hay periodos de prueba ni degradaciones.

## Cómo se activa Enterprise

Con una licencia JWT firmada por VA360 LABS, que se pega en **Administración → Licencia** o se fija en la variable `DIDACTA_LICENSE_KEY`. El detalle está en [Licencia Enterprise](../enterprise/license.md).
