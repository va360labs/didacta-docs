# Simple mode

**Simple mode** is a short way of using the panel, meant for people setting up their academy for the first time who want to go from zero to **selling** without touching anything technical. It is not another edition or another installation: it is a **per-academy switch** that folds away the advanced options. **Nothing is deleted**: everything in full mode is still there and you can go back to it whenever you like.

## Who has it on

- It is an academy setting (`simple_mode`) and **it is off by default**: a new installation starts in **full mode**, with the usual side menu.
- A platform that provisions academies for its creators (like Didacta Cloud, see [Your academy](../cloud/your-academy.md)) can create them already in simple mode.
- Whoever administers the academy turns it on and off without asking anyone:
    - **Turn on**: in full mode, **Configuration → "Volver al modo sencillo"** (Back to simple mode).
    - **Turn off**: in simple mode, **Settings → "Usar el modo completo"** (Use full mode) (**"Cambiar al modo completo"**).
    - Through the API: `GET`/`PUT /api/v1/admin/simple-mode` (`{ "enabled": true }`).

## What changes

| | Full mode | Simple mode |
| --- | --- | --- |
| Navigation | Side menu with all the groups | **Tabs on top**: Get started, Home, Community, Courses, Live, Members, Ranking, Students, Sales, Settings and "More" (everything else, with the same name as in the full menu) |
| First use | The [setup wizard](../installation/setup-wizard.md) and exploring | **A 3-question sign-up** and the "Get started" screen |
| Course editor, Settings, Sales, Students, Live, community and Referrals | Full screens | Simplified versions (the rest of the `/admin/…` and `/studio/…` routes still open from "More") |
| Avatar | Profile, certificates… | Plus "My progress" and **"Invite and earn"** (referrals) |

The **routes are the same** in both modes; a link to `/admin/payments` or `/studio/courses` works the same. The pages of the module catalog carry screenshots of full mode unless they say otherwise.

### The 3-question sign-up

The first time someone who administers an academy in simple mode comes in, they are asked three questions:

1. **What is your academy called?** (and the **academy language**: Spanish, English or Brazilian Portuguese; it can be changed in Settings).
2. **What are you going to sell?** A **course** (one-off payment) or a **membership** (monthly payment), with its title.
3. **How much does it cost?** Price and currency; the screen shows how much is left from each sale.

When finished it creates, **as a draft**, the course or the plan, the basic community spaces and a welcome post. It is idempotent: repeating it duplicates nothing. An academy that already had courses, students or sales before switching to simple mode is **adopted as it is**, with no sign-up.

### "Get started"

Six steps to the first sale, with the status of each:

1. **Your academy**: logo and colour.
2. **Your first course**: one lesson is enough (uploaded or linked video, or text).
3. **Price**: the one from sign-up is only a starting price; until it is confirmed the step does not count as done.
4. **Connect your payments**: with [platform payments](../modules/catalog/payouts.md), the "Connect payments" card (and then the tax details); without them, your own Stripe keys ([Payments](../modules/catalog/billing.md)).
5. **Publish**: generates the sales link (`/catalog/<slug>` or `/join`).
6. **Invite** your first students.

After the first sale the screen switches to a summary (sales, students, next live class).

## Where the advanced options went

Under **Settings → "Todas las opciones avanzadas"** (All advanced options: email, payments with your own account, Zoom, language and more), and **Settings → "Usar el modo completo"** brings the side menu back. The automated messages of [mod.retention](../modules/catalog/retention.md) and the rewards of [mod.gamification](../modules/catalog/gamification.md) are managed under **Settings** in simple mode.

## For the operator

- Which modules each academy has active does not change with the mode: it is independent of [Modules](../modules/managing.md).
- The mode can be set when creating an academy from instance administration (`simpleMode` when creating the tenant).
- It is a preference **of the academy**, not of each person: all administrators and instructors see the same.

!!! note "Labels"
    The quoted labels are the Spanish ones of the interface (the interface also exists in English and Brazilian Portuguese).
