# Limits and differences from Community

Cloud and Community are **the same product**: the same modules and the same published image. What changes is **who operates the installation** and, with it, what you can configure and how payments work. For the three editions, see [Editions](../getting-started/editions.md).

## What changes

| | Community (self-hosted) | Cloud |
| --- | --- | --- |
| **Who operates it** | You: server, database, backups, certificates. | VA360 LABS. |
| **Updates** | You apply them ([Upgrading](../installation/upgrading.md)). | Automatic; you do nothing. |
| **Cost** | Free (fair-code licence); you pay for your infrastructure. | Free; Didacta keeps [5% of your sales](pricing.md). |
| **Payments** | With your own Stripe account, with no Didacta fee. | Always through Didacta's Stripe account, which transfers your share every week and keeps 5%. You cannot connect your own Stripe account ([Payouts](payouts.md)). |
| **Countries you can get paid in** | Whatever your Stripe account allows. | 33 countries: the EU, Norway, Liechtenstein, the UK, Switzerland, the US and Canada. |
| **Academy mode** | Full (side menu), with the option to switch to simple. | Simple ("Start here"), with the option to switch to full. |
| **Address** | Your domain, as you configure it. | `<your-academy>.didacta.io`; no custom domain for now. |
| **Data** | Wherever you host it. | Servers in the EU. |
| **Backups** | Your own ([Backups](../installation/backups.md)). | Nightly and encrypted: 30 days and one per month for 12 months. |
| **Email** | Your SMTP. | The platform's service with your academy's name; own SMTP optional ([Your academy](your-academy.md#email)). |
| **Video** | Your storage or links. | Upload to Didacta with [limits](video-and-inactivity.md), or links. |
| **Academies with no activity** | No policy. | Archived after 30 days without activity. |
| **Stripe discount codes** | Work with your keys. | Switched off for now. |
| **Support** | The community (GitHub). | By email, with no guaranteed response times ([Support](support.md)). |

## What does not change

- **The product's modules**: courses, assessments, certificates, community, messaging, gamification, live classes, referrals, surveys… ([catalog](../modules/catalog/index.md)), except those listed below as not included.
- **AI with your own key**: if you use the tutor or the content generator, you connect your own OpenAI or Anthropic account.
- **The code is public**: what Cloud runs is what you can read and, if you want, install yourself.

## What Cloud does not include

- **White-label**: the academy footer always reads "Powered by Didacta".
- **SSO (SAML and OIDC) and SCIM**: these are Enterprise capabilities and Cloud does not include them.
- **Third-party modules** from the marketplace: on Cloud only the product's own modules run.
- **Migrator**: there is no migration tool on Cloud for now, neither from other platforms (the LearnDash Migrator module is not available) nor from Community ([Leaving and moving your data](data-and-migration.md)).
- **Your own Stripe account**: payments always go through Didacta.

## What you cannot configure on Cloud

Cloud does not expose the installation's configuration, because you do not operate it:

- **Environment variables** and everything that hangs from them (storage, database, secrets, global email).
- **Server or database access.**
- **A custom domain** and a **mobile app**: not included today.
- **Service-level agreement:** Didacta works to keep your academy always available, but today it does not guarantee an availability percentage or response times, and there may be maintenance interruptions (it gives notice whenever it can).

You can, in the [full mode](../installation/visual-walkthrough.md#10-simple-mode), use the product's advanced options (switchable modules, email templates, access groups, etc.).

## Third-party services you connect yourself

Some features use services you contract directly that are not part of Didacta Cloud: **Zoom** for live classes (or pasting a Zoom, Google Meet or Teams link) and **OpenAI or Anthropic** for the AI assistant. Didacta is not responsible for them.

## How to choose

- Choose **Cloud** if you want to open an academy today, with no server and no card, and you are in one of the [payout countries](payouts.md#countries-and-currencies).
- Choose **Community** if you need your own domain, your own infrastructure, to get paid from a country Cloud does not support, or to integrate with your systems with full control.
- You can **switch from one to the other**: [Leaving and moving your data](data-and-migration.md).
