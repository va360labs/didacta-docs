# mod.theming — Visual customization

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Visual branding **per organization**: logo (uploaded to storage or a URL), brand display mode (logo only / logo and name), favicon (uploaded or by URL), primary color expressed as an **HSL hue + saturation**, display and body font families, sanitised custom CSS, footer HTML and the headlines on the sign-in screens. It also defines how the academy is presented in its community: **cover**, **description** and **social links**.

## How it works

- The theme is loaded **server-side** in the web app's root layout and injects CSS variables that override the base design tokens: changing the hue propagates automatically to all 10 steps of the brand scale (`brand-50…900`) — there is no need to touch tokens one by one.
- Fonts are restricted to a **whitelist** of Google Fonts. Headlines: Sora, Inter, Manrope, Space Grotesk, DM Sans, Plus Jakarta Sans, Outfit and Lexend. Body: Inter, Manrope, DM Sans, IBM Plex Sans, Source Sans 3, Plus Jakarta Sans, Outfit and Nunito Sans.
- Custom CSS is **sanitised** and capped at 16 KB; the footer HTML accepts only basic tags.
- The logo, favicon and cover are served through **public** endpoints (necessary on `/signin` and in the community, before authentication), the logo with a 5-minute cache.
- The defaults are the Didacta identity (hue 213, saturation 70, Sora + Inter); `reset` returns to them.

!!! note "Which part is Enterprise"
    Basic branding (logo, colors, fonts, headlines) is **Community**. Only `customCss` and `footerHtml` with actual content require the `feat:white_label` capability (saving a logo and colors without a license always works). Hiding the Didacta brand is also white-label.

!!! note "Screenshots in full mode"
    The screenshots on this page come from **full mode** (side menu), the mode a new installation starts in. In **simple mode** the same routes are still available from "More", except for the screens that mode replaces with a simplified version (the course editor, Settings, Sales, Students, Live, the community and Referrals); to use them as shown here, switch to full mode under **Settings → "Use the full mode"**.

## Configuration

This is a **core** module: always active, it does not appear in the Modules tab switch. Everything is configured in **Admin → Branding** (`/admin/branding`), for `tenant_admin` and `super_admin` only, with a live **Preview** and a **Restore default values** button:

- **Brand color**: **Hue** (0–360) and **Saturation** (0–100) sliders, with the resulting `brand-50…900` scale in view, or the **Color code (hex)** field: paste your brand's (e.g. `#1E5AA8`) and hue and saturation are taken from it (each step's lightness is set by the scale).
- **Typography**: **Headline font (display)** and **Body font**, each with its whitelist.
- **Logo and favicon**: **Upload logo to storage** (PNG, JPG, SVG or WebP, 2 MB max; with **Replace logo** and **Delete**), or a **Logo URL (alternative)** (`https://…` or a relative `/api/v1/…` endpoint), plus the favicon: **Upload favicon** / **Replace favicon** (PNG, JPG, SVG or WebP, ideally square, 1 MB max; no ICO), **Use the logo as favicon**, or an https **Favicon URL**.
- **How should your brand be shown?** — the logo mode (`logo_display_mode`, added in alpha.113): **Logo only** (`logo_only`, shown whole at its own size; for logos that carry the name inside) or **Logo and name** (`logo_and_name`; for square or textless marks). The choice drives the classroom sidebar, `/signin` and the welcome assistant.

    ![The logo block of the Branding panel with the brand display mode selector](../../assets/modules/theming/en/theming-2-logo-mode.png)

- **Your academy in the community**: what members see on the academy card next to the feed. **Cover** (**Upload a cover**; JPG, PNG, WebP or SVG, up to 10 MB, a landscape image such as 1600 × 800; recompressed to WebP, except SVG; it replaces the color band), **Description** (up to 280 characters; if empty, the sign-in headline is used) and **Links** (5 at most: WhatsApp, Telegram, Instagram, YouTube, TikTok, Facebook, LinkedIn, X or website; `https://` only, and each network only accepts its own domains — a "WhatsApp" link pointing at another domain is rejected; pasting the link shows its network's icon).
- **Sign-in screen**: **Headline** (160 characters max) and **Supporting line (optional)** (240 max) for the `/signin` brand panel.
- **Custom CSS (advanced)** (16 KB max) and **Custom footer** (4 KB max): they require the Enterprise `feat:white_label` capability. Without a license an upgrade card is shown in their place, and the backend answers `402` to any attempt to save them with content — sending the fields empty or as `null` is always allowed.

The **Logo & color** step of the welcome assistant (`/welcome`) configures the same things in a minimal version: upload the logo (applied instantly), pick **How should we show it?** (**Logo only** / **Logo and name**) and the color from 10 swatches + a fine-tune bar; color and mode are saved on continuing. It is one of the assistant's two mandatory steps (together with "Name").

No environment variables.

## Step by step

1. Go to **Admin → Branding** (`/admin/branding`). The panel loads the tenant's current theme.
2. Move **Hue** and **Saturation**: the **Preview** column and the brand scale reflect the change instantly, without affecting other users yet.
3. Upload your logo with **Upload logo to storage** (or paste a **Logo URL (alternative)**) and choose between **Logo only** and **Logo and name** under **How should your brand be shown?**.
4. Adjust the fonts and, if you want, the sign-in screen's **Headline** and **Supporting line (optional)**.
5. Click **Save changes**: the theme is persisted and applied to the whole organization (other users see it on reload, because it is injected server-side).

    ![The Branding panel with the brand color and the live preview](../../assets/modules/theming/en/theming-1-branding-color.png)

6. (Enterprise) With a `feat:white_label` license, add **Custom CSS (advanced)** and a **Custom footer**.
7. **Restore default values** goes back to the Didacta identity in one click (it asks for confirmation).

For a brand-new academy, the `/welcome` assistant walks you through the same without entering the panel:

![The welcome assistant's "Logo & color" step with the classroom preview](../../assets/modules/theming/en/theming-3-welcome-brand.png)

The result shows where your students see it first, the sign-in screen:

![The sign-in screen with the tenant's logo, color and headline](../../assets/modules/theming/en/theming-4-signin.png)

In **simple mode**, the logo, the logo mode and the color live under **Settings → "Logo and colour"** (the same piece as the wizard, with its preview); alongside the academy's cover, description and links; the rest of the Branding panel stays under "More → Branding".

![Simple mode: Settings → "Logo and colour"](../../assets/modules/theming/en/theming-5-simple-settings.png)

## Dependencies

None.

## Data model

`mod_theming_tenant_theme` — a single row per tenant holding all the branding.

## API

Prefix `/modules/theming` (`me`, `me/reset`, `me/logo`, `me/favicon`, `me/cover`, and the public `tenants/:tenantId/logo`, `…/favicon` and `…/cover`). Details in [Reference → Community and people](../../api/reference/community.md#theming-modulestheming).

## Events

**Emits**: `theming.logo.uploaded` and `theming.favicon.uploaded` (when the logo or the favicon is uploaded; same payload: `tenantId`, `contentType` and `size`). It consumes none.
