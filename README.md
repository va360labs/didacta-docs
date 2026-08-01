# didacta-docs

Documentación oficial de **Didacta** (didacta.io) — LMS fair-code, modular y listo para Fundae.

Sitio generado con [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) (el mismo generador que usa docs.n8n.io) y publicado en **https://docs.didacta.io**.

## Desarrollo local

Requisitos: Python 3.11+.

```bash
# 1. Crear el entorno e instalar dependencias
python -m venv .venv
.venv/Scripts/activate        # Windows · en Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt

# 2. Servidor local con recarga en caliente
mkdocs serve                  # http://127.0.0.1:8000

# 3. Build estático (genera ./site)
mkdocs build --strict
```

## Estructura

- `mkdocs.yml` — configuración del sitio, tema y navegación. **Toda página nueva debe añadirse al `nav`.**
- `docs/` — contenido en Markdown, organizado por pestañas (Primeros pasos, Instalación, Configuración, Módulos, API, Enterprise, Comunidad).
- `docs/stylesheets/extra.css` — colores de marca. Cambia ahí la paleta si evoluciona el branding.
- `.github/workflows/deploy.yml` — build y despliegue automático a GitHub Pages en cada push a `main`.

## Publicación (docs.didacta.io)

El workflow de GitHub Actions construye el sitio y lo publica en GitHub Pages. Para servirlo bajo `docs.didacta.io`:

1. En el repo de GitHub: **Settings → Pages → Source: GitHub Actions**.
2. En el DNS de didacta.io: crea un `CNAME` de `docs` apuntando a `<org>.github.io`.
3. El fichero `docs/CNAME` ya fija el dominio personalizado; GitHub emite el certificado TLS automáticamente.

## Convenciones de contenido

- Todo en **español**, tono directo, segunda persona ("configura", "arranca").
- Los hechos técnicos (variables, rutas, comandos) salen del código real del repo `didacta-io` — nunca de memoria. Si el producto cambia, la página se actualiza en el mismo PR o en uno enlazado.
- Nada específico de un cliente concreto: dominios de ejemplo `example.com` / `ejemplo.com` y tenants `demo`.
- Admoniciones de Material (`!!! note`, `!!! warning`) para avisos; pestañas (`=== "Linux"`) para variantes por plataforma.

---

Copyright © 2026 VA360 LABS S.L. — la documentación se publica bajo los mismos términos fair-code que el producto ([didacta-io/LICENSE](https://github.com/va360labs/didacta-io/blob/main/LICENSE)).
