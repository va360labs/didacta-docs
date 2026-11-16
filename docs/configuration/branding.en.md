# Whitelabel branding

Didacta is **whitelabel out of the box**: nothing of a specific installation's brand lives in the code. Everything visual and textual on your platform is academy configuration.

## Branding per academy (Community)

Under **Administration → Branding** (`/admin/branding`) each academy configures:

- **Brand colour**: paste your brand's hex code (e.g. `#1E5AA8`) or move the hue and saturation; the 10 steps of the palette are derived automatically.
- **Typography**: headline font and body font, from a closed list of Google Fonts (Sora + Inter by default).
- **Logo and favicon**: uploaded to storage (PNG, JPG, SVG or WebP logo up to 2 MB; favicon up to 1 MB, or "use the logo as favicon") or an https URL is pasted. There are two presentations: **logo only** (for logos that already carry the name) or **logo and name**.
- **Sign-in screen** (`/signin`): headline (max. 160 characters) and supporting line (max. 240). The figures shown below (active students, published courses) are real and not configurable; they only appear once the academy has at least 10 students.
- **Restore defaults** puts the theme back to Didacta's.

Changes apply live in the preview and, on save, to **everyone in the academy**. This is done by the [`mod.theming`](../modules/catalog/theming.md) module, which also defines how the academy presents itself in the community (cover, description and links); its details are on its page.

!!! note "In simple mode"
    In [simple mode](simple-mode.md), **Settings → "Logo y color"** gathers the basics (name, logo and colour); the full screen can still be opened at its route, `/admin/branding`.

## Language

Each academy has a **language** (Spanish, English or Brazilian Portuguese) and each person can choose their own. See [Languages](languages.md).

## The academy's public page

What someone sees when they arrive **without a session** at the root of the academy's domain (`/`):

- **On**: a public front page with the name, cover, description, who runs it (name and photo of the first person who administers, never an email), the **number** of members (only with enough students; never their names or faces), the next live class (title and date, never the room link) and the **ways in**: courses on sale, membership and free access by request.
- **Off**: straight to the sign-in screen (`/signin`).

It is switched on and off from the academy settings (`PUT /api/v1/admin/tenant-settings/academy`, `publicHome` field). While nobody touches it, the mode decides: **on in simple mode** and **off in full mode**, so the entrance of already existing academies does not change. If the academy collects through [platform payments](../modules/catalog/payouts.md) and its account cannot sell yet, or sign-ups are frozen, the front page offers nothing for sale.

**Search engines**: the academy is only indexable with the public page on. Then the front page, the catalog (`/catalog`) and every published sales page (`/catalog/<slug>`) appear in `robots.txt` and `sitemap.xml`, with their JSON-LD; with it off, everything is closed (`noindex`). The canonical domain is the academy's **verified primary** one, if it is shareable (not `localhost`, an IP or `.test`); otherwise the request's.

## Own domain

The tenant is resolved by the request **host**: each academy has one or more associated domains (the first one is created in the [setup wizard](../installation/setup-wizard.md)).

- A `super_admin` adds or removes an academy's domains under **Administration → Tenants** (`POST`/`DELETE /api/v1/admin/tenants/:id/domains`); **the primary domain cannot be removed**.
- **A host that belongs to no academy answers 404**, and that of a suspended academy shows a page explaining it. It is an anti-phishing defence: with a DNS wildcard every free subdomain would be a working login. If your self-hosted installation needs to serve the app on any host, `DIDACTA_ALLOW_UNKNOWN_HOSTS=true` restores the previous behaviour.
- Resolution requires an **exact match**: there are no subdomain wildcards. Every subdomain you want to use must be registered.
- Putting the domain into production (DNS, TLS certificate and the proxy's routing to Didacta) is up to your infrastructure; see [docker compose](../installation/docker-compose.md).

### "Own domains" panel (Enterprise)

**Administration → Domains** (`/admin/domains`, capability `feat:custom_domains`) keeps the **record** of the academy's custom domains: adding one generates the CNAME target (the one in `DIDACTA_CNAME_TARGET`, `cname.didacta.io` by default) and a verification token; you configure the CNAME and TXT in your DNS and click **"Validar"** (Validate) with the token. A domain can be left `verified` or `inactive` without deleting it.

!!! warning "What the panel does not do yet"
    In this version verification is **manual**: it compares the token you paste and **does not query DNS**. And registering a domain here **does not route it**: traffic only reaches the academy if the host is registered on it (**Tenants**, above) and your proxy and certificate serve it. Without a licence the endpoint answers **402** and the screen shows what it unlocks.

## Full white-label (Enterprise)

Hiding the Didacta brand entirely ("powered by Didacta") and customising the **CSS** (max. 16 KB, sanitised: `@import`, `expression()`, `javascript:` and closing `<style>` are blocked) and the **HTML footer** (max. 4 KB, basic tags) is the Enterprise capability `feat:white_label`. Without a licence those cards show what they unlock and the backend answers **402** to any attempt to save them with content — like every Enterprise feature, it is [never hidden](../getting-started/editions.md). The rest of the branding (logo, colours, typography, texts) is Community and needs no licence.

!!! note "Labels"
    The quoted labels are the Spanish ones of the interface.
