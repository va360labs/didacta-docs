# Instalación y despliegue

Didacta Community se instala en tu propia infraestructura a partir de la imagen Docker oficial [`didactaio/community`](https://hub.docker.com/r/didactaio/community) (pública, no requiere `docker login`).

## Caminos de instalación

| Camino | Para quién | Guía |
| --- | --- | --- |
| **Docker Compose** (recomendado) | La mayoría de instalaciones: levanta app + Postgres + Redis + buzón de correo de pruebas de una vez. | [Instalar con Docker Compose](docker-compose.md) |
| **docker run manual** | Operadores que ya tienen Postgres 16 y Redis 7 administrados y solo quieren el contenedor de la aplicación. | [Instalar con docker run](docker-run.md) |

## Requisitos

- **Docker 24+** con Docker Compose v2 (`docker compose version`).
- **2 GB de RAM** libres y ~2 GB de disco para imágenes y datos.
- Puertos libres (todos configurables por variable de entorno): `3000` (web), `4000` (API), `5432` (Postgres), `6379` (Redis), `8025` (UI del buzón de correo de pruebas).

!!! warning "PostgreSQL con pgvector"
    El compose oficial usa la imagen `pgvector/pgvector:pg16`. Si traes tu propio Postgres tiene que ser **16+ con la extensión `pgvector` disponible** (el tutor IA guarda embeddings en columnas `vector`). Sin ella, la app no arranca (`type "vector" does not exist`).

## Solo 3 variables obligatorias

El resto de variables tienen valores por defecto razonables o las inyecta el compose. La referencia completa está en [Variables de entorno](../configuracion/variables-de-entorno.md).

| Variable | Qué es |
| --- | --- |
| `DATABASE_URL` | Connection string de Postgres 16 con `pgvector`. |
| `REDIS_URL` | Connection string de Redis 7. |
| `AUTH_SECRET` | Secreto para firmar sesiones y cookies. **Mínimo 32 caracteres** aleatorios. |

!!! tip "Generar `AUTH_SECRET`"
    Cualquier cadena aleatoria de 32+ caracteres sirve:
    ```bash
    openssl rand -base64 32
    # o
    node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
    ```
    Guárdala bien: si la cambias, todas las sesiones activas se invalidan.

## Versionado de la imagen

Didacta versiona con **SemVer** y publica cada versión como tag de imagen Docker. **No existe tag `latest`**: cada instalación fija su versión con `DIDACTA_IMAGE_TAG` en `.env` y decide cuándo subir. Los tags publicados están en [Docker Hub](https://hub.docker.com/r/didactaio/community).

## Después de instalar

1. [Asistente de configuración](setup-wizard.md) — crear la organización y el primer administrador.
2. [Configuración](../configuracion/index.md) — SMTP real, storage S3, branding, IA.
3. [Copias de seguridad](copias-de-seguridad.md) — antes de que haya datos de verdad.
4. [Actualizar](actualizacion.md) — cuando salga la siguiente versión.
