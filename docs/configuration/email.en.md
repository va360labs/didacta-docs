# Email

Didacta sends transactional email — invitations, password resets, class reminders, community digests, subscription notices — over SMTP, with **two levels of configuration**.

## 1. Instance-wide SMTP (environment variables)

This is the default transport for the whole instance. It requires all **three** variables together:

```bash
SMTP_HOST=smtp.your-provider.com
SMTP_PORT=587
SMTP_FROM="My Academy <no-reply@example.com>"
# Optional:
SMTP_USER=username
SMTP_PASS=password
SMTP_ENCRYPTION=starttls   # tls (465) | starttls (587) | none; empty = the port decides
```

If any of the first three is missing there is **no instance-wide SMTP**: only tenants that configure their own SMTP server will be able to send mail. `SMTP_SECURE` (`true`/`false`) is still accepted as a legacy variable; `SMTP_PASSWORD` is an alias of `SMTP_PASS`. Any SMTP provider works (for example, Amazon SES's SMTP credentials).

### Who signs the email with the shared SMTP

When an academy uses the instance-wide SMTP (the platform's), the sending **address** is always the one in `SMTP_FROM` (the one with SPF/DKIM), but the student sees the **academy's name**:

```text
From: "Academia Ñandú" <hola@example.com>
Reply-To: contacto@academia-nandu.com
```

Replies go to the academy's **"Reply-to email"** (see below) or, if it has none, to its **main administrator's** email (the academy's first active administrator). The name goes through a cleanup (max. 64 characters, no `<`, `>`, `@`, quotes or control characters) so it cannot be used to forge a header.

## 2. Per-tenant SMTP (admin panel)

Each organization can define its own server under **Administration → Settings → Notifications**. That setting takes priority over the instance-wide one, and its credentials are stored **encrypted at rest** in the database. From the same screen you can send a **test email** to check the configuration.

On the same tab, the **"Correo para respuestas"** (Reply-to email) card (`GET`/`PUT /api/v1/admin/tenant-settings/smtp/sender`) sets the `Reply-To` of the academy's emails, and shows **where replies go today** (with its own SMTP, to its `From`). Empty removes it and goes back to the main administrator.

## In development: Mailpit

The official compose file includes [Mailpit](https://mailpit.axllent.org/): every message from the installation lands in its web mailbox (`http://localhost:8025`) instead of going out to the internet. Ideal for testing complete flows without sending anything real.

!!! warning "Mailpit shows everything"
    The Mailpit UI shows **every** message, password resets included. That is why the compose file publishes it on `127.0.0.1` only. Never expose it.

## Links inside emails

The absolute links in emails (the password reset button, invitations…) are built from the academy's **verified primary domain** if it has one and, otherwise, from `WEB_PUBLIC_URL`. **With neither, emails in production carry links to `localhost`.** Always configure it:

```bash
WEB_PUBLIC_URL=https://campus.example.com
```

## Email templates (Administration → Emails)

The **content** of transactional emails is edited per tenant under **Administration → Emails**, with no code changes: subject and body accept variables (`{{tenantName}}`, `{{userName}}`…) and every template has a sensible default. The **structural parts** of each email (the action button, the OTP code, the approve/reject buttons) are not editable: they guarantee the flow keeps working even when the copy changes.

Core template keys:

| Key | When it is sent |
| --- | --- |
| `auth.password_reset` | Password reset. |
| `enrollment.welcome` | Account created by invitation/API, with a sign-in link. |
| `membership.welcome` | Account created by a membership purchase (`mod.subscriptions`). |
| `billing.welcome` | Account created by a public course purchase (`mod.billing`): "set your password". |
| `subscriptions.renewal_warning` | Upcoming renewal notice. |
| `payment_connections.access_expiring` | Access derived from an external payment account is about to expire. |
| `subscriptions.admin_digest` | Daily subscription summary for admins. |

Modules add their own to the same catalog — for example, `mod.member-registration` registers its 4 templates (`member_registration.otp_code`, `.approval_request`, `.welcome_approved`, `.rejection`) — and the hub notifications (enrolment, live classes, surveys, referrals, help notices, automated messages…) are on the same screen. The emails of each sale have their own templates (see [mod.billing → Sales emails](../modules/catalog/billing.md)). Templates exist in Spanish, English and Portuguese.

!!! note "Language of what you customise"
    Transactional templates (password reset, welcomes, renewal notices…) are customised on the Email channel **in Spanish (es-ES)**; hub ones let you pick channel and language in the editor. See also [Languages](languages.md). The read/edit endpoints are in [Reference → Administration](../api/reference/administration.md).

## If no SMTP is configured

Emails are not sent but they **are written to the logs** — the platform does not block. Even so, flows such as user invitations and password resets depend on email: configure it before you onboard real users.
