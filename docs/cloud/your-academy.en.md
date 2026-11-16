# Your academy

What you can customise in your Didacta Cloud academy and what you cannot. In simple mode the basics are under **Settings**; the rest is under **Settings → All advanced options** (or in the [full mode](../installation/visual-walkthrough.md#10-simple-mode)).

## Address

Your academy lives at `https://<your-academy-name>.didacta.io`. That address is derived from the name you enter when signing up: lowercase, without accents and with hyphens in place of spaces and symbols (for example, "Watercolour Workshop" becomes `watercolour-workshop`). If it is already taken a number is added, and some names are reserved and never granted (`www`, `api`, `admin`, `login`…). You see it in the second question of the sign-up ("Your address").

!!! note "Custom domain"
    You cannot use your own domain on Didacta Cloud today. You can add your logo and your colour.

## Logo and colour

In **Settings → Logo and colour** (in full mode, **Administration → Branding**) you upload your logo and choose your colour. Your students see them in the header, on your sales page and at checkout.

- **Without a logo**, the academy uses **your initials** on your colour.
- Payment happens inside your sales page, with your logo (or your initials) and your colour. The receipts and emails your buyers get carry your branding, and you edit their text in **Administration → Emails**.
- The academy footer reads "Powered by Didacta". Hiding the Didacta brand is the Enterprise [white-label](../configuration/branding.md#full-white-label-enterprise) capability.

## Language

Your academy can run in **Spanish, English or Brazilian Portuguese**. The academy's language is:

- the language of your sales page and your catalog;
- the language of emails to people who have not picked one yet;
- the language of the texts the system writes for everyone (for example, the announcement of a live class in the community).

It starts in the language you signed up from (`/es`, `/en` or `/pt-br`). To change it: **Settings → Academy language** (in full mode, **Administration → Settings → General**). **Each student can choose their own** in their profile.

## Email

Your academy's emails (notices, password recovery, notifications, purchase receipts) go out through the **platform's email service**, with **your academy's name** as the sender. You configure nothing for them to arrive.

| What | Where |
| --- | --- |
| **Where replies go.** If a student replies to an email, the reply reaches the "Reply-to email" you set. If you set none, it reaches the email of the person who administers the academy. | **Administration → Settings → Notifications** (`/admin/settings?tab=notifications`) |
| **Send from your own email service.** If you prefer emails to go out through your SMTP, connect it in the same tab and send a test email. | Same tab. See [Email](../configuration/email.md) |
| **The text of each email.** You change the subject and body of the emails the platform sends. | **Administration → Emails** |

!!! tip "Set a reply-to email"
    It is a single field and it stops your students' replies ending up in a mailbox you do not check.

## What you cannot change on Cloud

Cloud does not expose instance configuration: there are no environment variables, no storage of your own and no database access, because you do not operate the server. The full list is in [Limits and differences from Community](cloud-vs-community.md).
