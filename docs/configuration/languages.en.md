# Languages

Didacta's interface, its emails and the texts the server writes on your behalf exist in **three languages**: **Spanish**, **English** and **Brazilian Portuguese**. The content you write yourself (courses, lessons, posts…) is always in the language you write it in: it is not translated automatically.

## Two levels

| | What it is | Where it is chosen |
| --- | --- | --- |
| **Academy language** | The one of the sales page and catalog, of emails to people who have not chosen a language yet, of the texts the server writes for everyone (the announcement of a live class in the community, the community's initial spaces…) and what someone sees when they arrive with no cookie or profile. | In the [setup wizard](../installation/setup-wizard.md) (it proposes your browser's language if there is a matching one), in the **3-question sign-up** of [simple mode](simple-mode.md) and under **Settings → General** (`PUT /api/v1/admin/tenant-settings/academy`, `locale` field: `es`, `en` or `pt`). A `tenant_admin` or `super_admin` changes it. |
| **Each person's language** | The one of their interface and of their emails and notices. | In their profile (`/account`) or in the language selector. It is a personal preference. |

### Which language is shown

The interface is rendered with the first of these that exists: the **`didacta_locale` cookie** (what the visitor chose in the selector) → the person's **profile language** → the **academy language** → Spanish. On public pages (such as the checkout without an account) the cookie wins. Emails go in the **recipient's profile language**; if they do not have one yet, in the academy's.

!!! info "Variants"
    The profile stores `es-ES`, `es-AR`, `en-US` or `pt-BR`. `es-AR` uses the same Spanish catalogue and only changes the date and number format. Any other Portuguese (for example `pt-PT`) is served as `pt-BR`, and any other English as `en-US`.

## What is translated

- The **interface**, entirely, with a CI guard (`messages-parity`) that stops a language from shipping half-done.
- **Emails and notices**: the notification-hub templates (purchase welcomes, reminders, digest, subscriptions…) are in the three languages; each template can be **customised per academy** under **Administration → Emails** (see [Email](email.md)). The buyer of an anonymous checkout receives their emails in the language they were browsing in.
- **Compliance texts**: the waiver of the right of withdrawal, the PDF certificate (es, en, pt) and the `.ics` files of live classes.
- **Simple-mode onboarding and automated messages** of [mod.retention](../modules/catalog/retention.md): templates written in the academy's language, with variables valid in all three languages.

!!! note "What is not included"
    Texts written by the teams (courses, lessons, posts, hand-edited templates) stay as written. The admin panel offers no languages beyond these three; a new language requires a complete message catalogue in the code.
