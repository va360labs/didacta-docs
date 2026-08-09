# Didacta Enterprise

Enterprise adds to Community a closed set of **cross-cutting core capabilities** aimed at large organisations: identity federation, automatic provisioning, full white-label, extended auditing…

The split is simple and does not change: **modules are always Community**; the only paid features are these platform capabilities. See [Editions](../primeros-pasos/ediciones.md).

## What it includes

| Area | Capabilities |
| --- | --- |
| Corporate identity | SAML 2.0 SSO · OIDC SSO · SCIM provisioning · mandatory MFA per organization |
| Multi-organization platform | Real multi-tenancy (several isolated organizations on one instance) · per-tenant custom domains |
| Branding | Full white-label (hiding the Didacta brand) |
| Compliance | Signed 7-year audit retention · cryptographically signed XLSX/PDF exports |
| API | High-throughput webhooks (queue, HMAC, dead-letter) · elevated API rate limits |

The details of each one, with its exact effect and how it compares to Community: [Capabilities](capabilities.md).

## How it behaves without a license

In Didacta, **gating never hides anything**:

- Enterprise screens always exist, with their title and description; without a license they show a notice explaining what they unlock, instead of the panel.
- Enterprise endpoints return **HTTP 402** with the required capability in the body.
- Community works **in full** without a license: no user limits, no trials, no degradation.

## How it is activated

With a JWT license signed by VA360 LABS, pasted into **Administration → License** or pinned in the `DIDACTA_LICENSE_KEY` variable (the environment wins over the panel). The full behaviour (states, grace period, verification) is described in [License](licencia.md).

## Purchasing

Enterprise also includes a dedicated account manager, guided onboarding and integrations with your existing systems. Information and contact at [didacta.io](https://didacta.io) · `licensing@didacta.io`.
