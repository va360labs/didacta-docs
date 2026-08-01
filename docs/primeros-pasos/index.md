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

Didacta está en **alpha**: entre mayo y julio de 2026 el producto maduró sirviendo en producción real a su primer despliegue; desde el 31 de julio de 2026 el repositorio es el producto whitelabel y prepara su primera versión pública.

- Versionado **SemVer**; cada versión se publica como tag de imagen Docker. **No existe tag `latest`**: fija siempre una versión concreta.
- Historial de cambios en el [CHANGELOG](https://github.com/va360labs/didacta-io/blob/main/CHANGELOG.md).

## Siguientes pasos

- [Ediciones: Community, Enterprise y Cloud](ediciones.md)
- [Arquitectura de la plataforma](arquitectura.md)
- [Instalar Didacta en 5 minutos](../instalacion/docker-compose.md)
