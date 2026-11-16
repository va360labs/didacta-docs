# Editions: Community, Enterprise and Cloud

Didacta is **one product with three editions**. The code is the same; what changes is who operates it and which cross-cutting capabilities are active.

| Edition | Who it is for | What it includes |
| --- | --- | --- |
| **Community** | Teams that deploy and operate the platform themselves. | The full source code and **every module**, with no user limit. Free, self-hosted, under the Sustainable Use License. |
| **Enterprise** | Organisations that need an SLA, custom integrations and a certified partner. | Community + cross-cutting core capabilities (SAML/OIDC SSO, SCIM, white-label…), unlocked with a signed license. |
| **Cloud** | Anyone who wants to open an academy in minutes, with no infrastructure. | The same product, hosted and maintained by VA360 LABS, with hands-off upgrades. Free: Didacta keeps 5% of each sale you make. See [Didacta Cloud](../cloud/index.md). |

Pricing and sales: [didacta.io](https://didacta.io). Cloud needs no purchase: you sign up at [didacta.io/en/start](https://didacta.io/en/start) and [this section](../cloud/index.md) explains how it works.

## Where the line is drawn

The line separating Community from Enterprise is deliberately simple:

- **Modules are always Community.** No module — courses, certificates, gamification, Fundae, virtual classroom… — is ever locked behind a license. What a self-hoster installs is the complete product.
- **The only paid features are cross-cutting core capabilities**: a closed list of platform features (identity federation, provisioning, white-label…) aimed at large organisations. The full list is in [Enterprise capabilities](../enterprise/capabilities.md).

## Where Cloud fits

Cloud is not a separate product line: it runs **the same published image** that is installed in Community, with the same modules. What changes is who operates it —VA360 LABS— and how payments work: your students' payments are received by Didacta, which transfers your share to you and keeps 5% of each sale. What you can and cannot configure, and how to move from one edition to the other, is in [Limits and differences from Community](../cloud/cloud-vs-community.md) and [Leaving and moving your data](../cloud/data-and-migration.md).

## What gating looks like in the product

In Didacta **nothing is hidden**.

- Every Enterprise screen **always exists** in the menu and loads with its title and description; if there is no license, the panel shows an upsell notice instead of the content.
- In the API, Enterprise endpoints respond with **HTTP 402** (`Payment Required`) when the capability is not active.
- A Community installation **works in full** without a license: there are no trial periods and no degraded behaviour.

## How Enterprise is activated

With a JWT license signed by VA360 LABS, pasted into **Administration → License** or pinned in the `DIDACTA_LICENSE_KEY` variable. The details are in [Enterprise license](../enterprise/license.md).
