---
hide:
  - navigation
  - toc
---

<p class="didacta-hero">
  <img src="../assets/logo.png#only-light" alt="Didacta — a modular, open learning platform built to scale">
  <img src="../assets/logo-blanco.png#only-dark" alt="Didacta — a modular, open learning platform built to scale">
</p>

# Didacta documentation

**Didacta** is a next-generation fair-code LMS (Learning Management System): source available, modular architecture, no per-user licensing, and legal compliance (Fundae, GDPR, WCAG 2.2 AA) built into the core.

This documentation covers everything you need to install, operate and extend your own **Didacta Community** instance, plus what the **Enterprise** and **Cloud** editions add on top.

<div class="didacta-cards">
  <a href="primeros-pasos/"><h3>🚀 Getting started</h3><p>What Didacta is, how the product is organised and its three editions.</p></a>
  <a href="instalacion/"><h3>📦 Installation and deployment</h3><p>Docker Compose in 5 minutes, manual docker run, setup wizard, upgrades and backups.</p></a>
  <a href="configuracion/"><h3>⚙️ Configuration</h3><p>One-to-one reference for every environment variable, plus database, storage, email, AI and branding.</p></a>
  <a href="modulos/"><h3>🧩 Modules</h3><p>The full catalog with one page per module, how to manage them and the guide to building your own.</p></a>
  <a href="api/"><h3>🔌 API</h3><p>Authentication, multi-tenant conventions and an endpoint-by-endpoint reference of the REST API.</p></a>
  <a href="enterprise/"><h3>🏢 Enterprise</h3><p>Enterprise capabilities, how licensing works and how to activate it.</p></a>
</div>

## Quick paths

- **I want to try Didacta right now** → [Installing with Docker Compose](instalacion/docker-compose.md)
- **I came here to upgrade my instance** → [Upgrading Didacta](instalacion/actualizacion.md)
- **I am looking for a specific environment variable** → [Environment variables](configuracion/variables-de-entorno.md)
- **I want to build a module** → [Building a module](modulos/crear-un-modulo/index.md)
- **I want to contribute to the project** → [Contributing](comunidad/contribuir.md)

## About the project

| | |
| --- | --- |
| **Website and demo** | [didacta.io](https://didacta.io) |
| **Source code** | [github.com/va360labs/didacta-io](https://github.com/va360labs/didacta-io) |
| **Docker image** | [ghcr.io/va360labs/didacta-community](https://github.com/va360labs/didacta-io/pkgs/container/didacta-community) (public) |
| **License** | [Didacta Sustainable Use License v1.0](https://github.com/va360labs/didacta-io/blob/main/LICENSE) (fair-code) |
| **Status** | Alpha — SemVer versioning, no `latest` tag |

!!! info "Fair-code"
    Didacta is **source-available with free internal use**: audit it, modify it and deploy it for your own organisation at no cost. Commercial distribution, third-party SaaS and white-labelling require an agreement with VA360 LABS S.L. Details in [Licenses](comunidad/licencias.md).
