---
render_macros: true
---

# Qué es Didacta

Didacta es un **LMS fair-code de nueva generación** para academias, formadores y organizaciones que quieren operar su propia plataforma de formación con control total.

## Los cuatro pilares

**Modular de verdad.**
Cada función es un módulo limpio: cursos, evaluaciones, certificados, gamificación, comunidad, aula virtual… Instala solo lo que necesitas, sin parches ni temas que rompen en cada actualización. La arquitectura de módulos está descrita en [Módulos](../modulos/index.md).

**Fair-code.**
Tu plataforma, tu código. Audítalo, modifícalo y despliégalo con uso interno libre bajo la [Didacta Sustainable Use License v1.0](https://github.com/va360labs/didacta-io/blob/main/LICENSE). Sin licencias por usuario. La distribución comercial, el SaaS o el white-label de terceros requieren acuerdo (ver [Licencias](../comunidad/licencias.md)).

**Cumplimiento serio.**
Fundae, RGPD y WCAG 2.2 AA integrados en el núcleo, no añadidos con plugins de terceros. Trazabilidad, auditoría y exportación de datos desde el día uno.

**IA discreta.**
Inteligencia artificial que ayuda sin interrumpir: crea contenido, sugiere itinerarios y resume actividad. El proveedor lo eliges tú: la capa de IA es **BYOK multi-proveedor** — cada instalación configura su proveedor y su clave (ver [Configurar la IA](../configuracion/ia.md)).

## Tres formas de llenar tu academia de alumnos

Los tres caminos de captación conviven en la misma instalación y se combinan libremente:

1. **Tú los das de alta.** Invita alumnos uno a uno o por lotes desde el panel: eliges su [grupo de acceso](../modulos/catalogo/access-groups.md) al invitarles, reciben un email para crear su contraseña y entran directamente a sus cursos. Desde la ficha de cada alumno gestionas sus grupos, matrículas y bajas.

2. **Vendes cursos sueltos.** Publica un curso, ponle precio y comparte tu catálogo público en `/catalogo`. El visitante paga con tarjeta vía Stripe sin registrarse antes: su cuenta se crea automáticamente con el email confirmado en el pago y queda matriculado al instante ([mod.billing](../modulos/catalogo/billing.md)). Los reembolsos retiran el acceso solos.

3. **Vendes membresías.** Crea planes con la periodicidad (1–12 meses) y la moneda que quieras, con periodo de prueba opcional. Tu página pública de venta en `/unete` muestra el catálogo real; al suscribirse, el alumno accede a todos los cursos del grupo que definas ([mod.subscriptions](../modulos/catalogo/subscriptions.md)). Si deja de pagar, el acceso se revoca automáticamente — sin tocar lo concedido a mano.

Y si tu comunidad exige aprobación previa, activa el [registro con solicitud](../modulos/catalogo/member-registration.md): verificadores configurables por tenant, evidencia de cada solicitud y decisión en un clic.

## Stack tecnológico

| Capa | Tecnología |
| --- | --- |
| Backend | Node.js 22 · NestJS 11 · TypeScript |
| Frontend | Next.js 15 (App Router) · React 19 · Tailwind · shadcn/ui |
| Base de datos | PostgreSQL 16 + Row-Level Security · Prisma · pgvector |
| Cache / colas | Redis 7 · BullMQ |
| Object storage | S3-compatible (MinIO en desarrollo, cualquier proveedor S3 en producción) |
| Aula virtual | Zoom (API + SDK Web) |
| Monorepo | Turborepo · pnpm workspaces |

## Estado del proyecto

Didacta está en **{{ didacta_channel|lower }}** ({{ didacta_version }}): entre mayo y julio de 2026 el producto maduró sirviendo en producción real a su primer despliegue; desde el 31 de julio de 2026 el repositorio es el producto whitelabel, y en agosto de 2026 entró en beta pública.

- Versionado **SemVer**; cada versión se publica como tag de imagen Docker. **No existe tag `latest`**: fija siempre una versión concreta.
- Historial de cambios en el [CHANGELOG](https://github.com/va360labs/didacta-io/blob/main/CHANGELOG.md).

## Siguientes pasos

- [Ediciones: Community, Enterprise y Cloud](ediciones.md)
- [Arquitectura de la plataforma](arquitectura.md)
- [Instalar Didacta en 5 minutos](../instalacion/docker-compose.md)
