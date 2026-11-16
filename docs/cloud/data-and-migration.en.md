# Leaving and moving your data

Your content and your students' data are yours. This page explains how you take them with you, what happens if you leave, and how to move from Didacta Cloud to your own installation (Community) and back.

## Taking your data with you

Today there is **no button to download your whole academy**. You ask for it by email:

1. Write to [hola@didacta.io](mailto:hola@didacta.io) asking for a copy of your academy.
2. Within a maximum of **30 days** and **at no cost**, Didacta hands it over in common formats:
    - spreadsheets with your **students**, their **enrolments**, your **sales** and your **community posts**;
    - the **texts and files of your courses**;
    - the **videos you uploaded** to Didacta.

What you can export yourself, whenever you want, from **Sales**: sales as CSV or Excel and buyers as CSV (with the filters you have set). It is personal data: keep it carefully.

## Leaving

- You can end the contract **whenever you want and at no cost**, by writing to [hola@didacta.io](mailto:hola@didacta.io).
- **When it ends** you can no longer sell. Your students' live subscriptions are cancelled **at the end of the period they have already paid**, with no further charges.
- Your **Stripe payout account stays open for 120 days** so refunds and disputes can be handled. After that period, and once debts are settled, the remaining balance is paid to you, including the minimum balance.
- Your academy is **kept for 30 days** so you can ask for your copy. After that it is deleted, except for what the law requires to be kept (for example, the record of sales and fees, 6 years).
- If Didacta wants to end the contract, it gives you **at least 30 days' notice** and tells you why (except for serious causes, explained in the [terms](https://didacta.io/en/legal/terms), section 13). If it stopped offering Didacta Cloud to everyone, it would also give you at least 30 days' notice.

!!! info "Academies archived for inactivity"
    If your academy was archived for [having no activity](video-and-inactivity.md#academies-with-no-activity), your students and your sales history are still there: write to the same address to reopen it.

## From Cloud to Community (self-hosted)

Cloud runs the **same published image** that is installed in Community, so everything you have in your academy also exists in your own installation. What does not exist is a migrator: Cloud has no migration tool for now, so the transfer is manual. The route:

1. **Ask for the copy** of your academy (above).
2. **Install Community** with [Docker Compose](../installation/docker-compose.md) and go through the [setup wizard](../installation/setup-wizard.md).
3. **Recreate your content** from the texts and files in the copy: courses, lessons, community.
4. **Bring your students**: invite them from the panel, one by one or in batches. They get an email to set their password.
5. **Connect your payments**: in Community you get paid with your own Stripe account, without Cloud's 5% fee. See [Payment connections](../modules/catalog/payment-connections.md) and [Billing](../modules/catalog/billing.md).

!!! warning "Paid memberships do not transfer by themselves"
    On Cloud your students' payments are collected in Didacta's Stripe account, and when you leave the subscriptions are cancelled at the end of the paid period. Students with a membership will have to subscribe again on your new installation. If you like, give them access by hand on your installation during the transition.

## From Community to Cloud

Cloud creates a **new academy** at [didacta.io/en/start](https://didacta.io/en/start). Cloud has no migrator for now: there is no wizard that moves a whole Community installation or imports data from other platforms, and database backups restore *that* installation, they are not meant to be imported into Cloud. The route:

1. Open your Cloud academy and go through [Getting started](getting-started.md).
2. Recreate your courses and bring your students by invitation or CSV.
3. For students who already paid you on your installation, you can give them **access by hand** without them paying again and without any fee being generated: it is a permitted use for students who already bought from you elsewhere. What is not allowed is using it as a regular way to sell outside Didacta.
4. Connect your [payouts](payouts.md) for new sales. If you prefer, you can go in stages: start your next course or group on Cloud and leave the rest where it is.

## If Didacta shut down

Didacta's code is public: you can install it on your own server and carry on. Cloud and Community are the same product, which is why the [Cloud to Community](#from-cloud-to-community-self-hosted) route exists.
