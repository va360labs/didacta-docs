# Whitelabel branding

Didacta is **whitelabel out of the box**: nothing about a particular installation's brand lives in the code. Everything visual and textual about your platform is tenant configuration.

## Branding per organization (Community)

Under **Administration → Branding**, each organization configures:

- Logo and favicon.
- Primary color (the theme derives the rest of the palette from it).
- Sign-in screen copy (headline and subtitle).

The `mod.theming` module extends visual customization further: fonts, design tokens and sanitised custom CSS.

## Custom domain

The tenant is resolved from the request **host**: each organization has one or more verified domains associated with it (the first one is created in the [setup wizard](../instalacion/setup-wizard.md)).

- Administrators manage additional domains from the panel (**Administration → Tenants**).
- Advanced management of **per-tenant custom domains** with CNAME verification is an Enterprise capability (`feat:custom_domains`). The UI shows the CNAME target, which is configurable with `DIDACTA_CNAME_TARGET`.

!!! note "No subdomain wildcards"
    Tenant resolution requires an **exact domain match** (an anti-phishing decision): every subdomain you want to use must be registered explicitly.

## Full white-label (Enterprise)

Hiding the Didacta brand entirely ("powered by Didacta") is the `feat:white_label` Enterprise capability. Without a license the white-label screen still exists and explains what it unlocks — like every Enterprise feature, [it is never hidden](../primeros-pasos/ediciones.md). The rest of the branding (logo, colors, copy) is Community and requires no license.
