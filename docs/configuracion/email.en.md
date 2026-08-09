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
SMTP_SECURE=true      # forces implicit TLS; if unset, the port decides
```

If any of the first three is missing there is **no instance-wide SMTP**: only tenants that configure their own SMTP server will be able to send mail.

## 2. Per-tenant SMTP (admin panel)

Each organization can define its own server under **Administration → Settings → Notifications**. That setting takes priority over the instance-wide one, and its credentials are stored **encrypted at rest** in the database.

## In development: Mailpit

The official compose file includes [Mailpit](https://mailpit.axllent.org/): every message from the installation lands in its web mailbox (`http://localhost:8025`) instead of going out to the internet. Ideal for testing complete flows without sending anything real.

!!! warning "Mailpit shows everything"
    The Mailpit UI shows **every** message, password resets included. That is why the compose file publishes it on `127.0.0.1` only. Never expose it.

## Links inside emails

The absolute links in emails (the password reset button, invitations…) are built from `WEB_PUBLIC_URL`. **Without that variable, emails in production carry links to `localhost`.** Always configure it:

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

Modules add their own to the same catalog — for example, `mod.member-registration` registers its 4 templates (`member_registration.otp_code`, `.approval_request`, `.welcome_approved`, `.rejection`). The read/edit endpoints are in [Reference → Administration](../api/referencia/administracion.md).

## If no SMTP is configured

Emails are not sent but they **are written to the logs** — the platform does not block. Even so, flows such as user invitations and password resets depend on email: configure it before you onboard real users.
