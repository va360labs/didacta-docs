---
hide:
  - navigation
  - toc
---

<p class="didacta-hero">
  <img src="assets/logo.png#only-light" alt="Didacta — Plataforma educativa modular, abierta y preparada para escalar">
  <img src="assets/logo-blanco.png#only-dark" alt="Didacta — Plataforma educativa modular, abierta y preparada para escalar">
</p>

# Documentación de Didacta

**Didacta** es un LMS (Learning Management System) fair-code de nueva generación: código fuente disponible, arquitectura modular, sin licencias por usuario y con cumplimiento legal (Fundae, RGPD, WCAG 2.2 AA) integrado en el núcleo.

Esta documentación cubre todo lo necesario para instalar, operar y extender tu propia instancia de **Didacta Community**, y para entender qué añaden las ediciones **Enterprise** y **Cloud**.

<div class="didacta-cards" markdown>

<a href="primeros-pasos/">
<h3>🚀 Primeros pasos</h3>
<p>Qué es Didacta, cómo se organiza el producto y sus tres ediciones.</p>
</a>

<a href="instalacion/">
<h3>📦 Instalación y despliegue</h3>
<p>Docker Compose en 5 minutos, docker run manual, asistente de configuración, actualizaciones y copias de seguridad.</p>
</a>

<a href="configuracion/">
<h3>⚙️ Configuración</h3>
<p>Referencia 1 a 1 de todas las variables de entorno, base de datos, almacenamiento, email, IA y branding.</p>
</a>

<a href="modulos/">
<h3>🧩 Módulos</h3>
<p>Catálogo completo de módulos, cómo gestionarlos y la guía para crear el tuyo.</p>
</a>

<a href="api/">
<h3>🔌 API</h3>
<p>Autenticación, convenciones multi-tenant y referencia de la API REST.</p>
</a>

<a href="enterprise/">
<h3>🏢 Enterprise</h3>
<p>Capabilities Enterprise, cómo funciona la licencia y cómo activarla.</p>
</a>

</div>

## Rutas rápidas

- **Quiero probar Didacta ya** → [Instalación con Docker Compose](instalacion/docker-compose.md)
- **Vengo a actualizar mi instancia** → [Actualizar Didacta](instalacion/actualizacion.md)
- **Busco una variable de entorno concreta** → [Variables de entorno](configuracion/variables-de-entorno.md)
- **Quiero desarrollar un módulo** → [Crear un módulo](modulos/crear-un-modulo/index.md)
- **Quiero contribuir al proyecto** → [Contribuir](comunidad/contribuir.md)

## Sobre el proyecto

| | |
| --- | --- |
| **Web y demo** | [didacta.io](https://didacta.io) |
| **Código fuente** | [github.com/va360labs/didacta-io](https://github.com/va360labs/didacta-io) |
| **Imagen Docker** | [didactaio/community](https://hub.docker.com/r/didactaio/community) (pública) |
| **Licencia** | [Didacta Sustainable Use License v1.0](https://github.com/va360labs/didacta-io/blob/main/LICENSE) (fair-code) |
| **Estado** | Alpha — versionado SemVer, sin tag `latest` |

!!! info "Fair-code"
    Didacta es **source-available con uso interno libre**: audítalo, modifícalo y despliégalo para tu organización sin coste. La distribución comercial, el SaaS de terceros o el white-label requieren acuerdo con VA360 LABS S.L. Detalles en [Licencias](comunidad/licencias.md).
