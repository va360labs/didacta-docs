# mod.theming — Personalización visual

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Branding visual **por organización**: logo (subido al storage o URL), favicon, color primario expresado como **hue + saturación HSL**, familias tipográficas display y body, CSS custom sanitizado, footer HTML y los titulares de las pantallas de acceso.

## Cómo funciona

- El theme se carga **server-side** en el layout raíz de la web e inyecta variables CSS que sobrescriben los design tokens base: cambiar el hue propaga automáticamente a los 10 escalones de la escala de marca (`brand-50…900`) — no hay que tocar token a token.
- Las fuentes están limitadas a una **whitelist** de Google Fonts (Sora, Inter, Manrope, Space Grotesk, DM Sans, Plus Jakarta Sans…).
- El CSS custom se **sanitiza** y se limita a 16 KB; el footer HTML solo admite etiquetas básicas.
- El logo se sirve por un endpoint **público** (necesario en `/signin`, antes de autenticar) con caché de 5 minutos.
- Los defaults son la identidad Didacta (hue 213, saturación 70, Sora + Inter); `reset` vuelve a ellos.

!!! note "Qué parte es Enterprise"
    El branding básico (logo, colores, fuentes, titulares) es **Community**. Solo `customCss` y `footerHtml` con contenido requieren la capability `feat:white_label` (guardar logo y colores sin licencia funciona siempre). Ocultar la marca Didacta también es white-label.

## Dependencias

Ninguna.

## Modelo de datos

`mod_theming_tenant_theme` — un único registro por tenant con todo el branding.

## API

Prefijo `/modules/theming` (`me`, `me/reset`, `me/logo`, logo público). Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#theming-modulestheming).

## Eventos

**Emite**: `theming.logo.uploaded`. No consume.

## Configuración

Todo por tenant en la propia tabla; sin variables de entorno.
