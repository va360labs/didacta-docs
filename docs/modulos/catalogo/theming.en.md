# mod.theming — Visual customization

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Visual branding **per organization**: logo (uploaded to storage or a URL), favicon, primary color expressed as an **HSL hue + saturation**, display and body font families, sanitised custom CSS, footer HTML and the headlines on the sign-in screens.

## How it works

- The theme is loaded **server-side** in the web app's root layout and injects CSS variables that override the base design tokens: changing the hue propagates automatically to all 10 steps of the brand scale (`brand-50…900`) — there is no need to touch tokens one by one.
- Fonts are restricted to a **whitelist** of Google Fonts (Sora, Inter, Manrope, Space Grotesk, DM Sans, Plus Jakarta Sans…).
- Custom CSS is **sanitised** and capped at 16 KB; the footer HTML accepts only basic tags.
- The logo is served through a **public** endpoint (necessary on `/signin`, before authentication) with a 5-minute cache.
- The defaults are the Didacta identity (hue 213, saturation 70, Sora + Inter); `reset` returns to them.

!!! note "Which part is Enterprise"
    Basic branding (logo, colors, fonts, headlines) is **Community**. Only `customCss` and `footerHtml` with actual content require the `feat:white_label` capability (saving a logo and colors without a license always works). Hiding the Didacta brand is also white-label.

## Dependencies

None.

## Data model

`mod_theming_tenant_theme` — a single row per tenant holding all the branding.

## API

Prefix `/modules/theming` (`me`, `me/reset`, `me/logo`, public logo). Details in [Reference → Community and people](../../api/referencia/comunidad.md#theming-modulestheming).

## Events

**Emits**: `theming.logo.uploaded`. It consumes none.

## Configuration

Everything is per tenant in its own table; no environment variables.
