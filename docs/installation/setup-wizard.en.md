# Setup wizard

Didacta's first start **uses no data seeds**: the instance is configured through the web wizard (`/setup`), which creates the organization and the first administrator account.

## When it appears

While the instance has no tenant at all, the web app redirects **every route** to the wizard. As soon as the wizard finishes, it is gone for good: the initialization endpoint returns `409 ALREADY_INITIALIZED` if an organization already exists.

The wizard is protected by a **single-use token**: when it starts with no organization, the instance prints it to its logs and the wizard can only be completed by opening `/setup?token=…` (`docker logs didacta-app | grep "Setup token"`; `install.sh` already shows the full link). That way nobody who reaches a freshly published instance before you can take the `super_admin` account. If the instance restarts before you finish, it prints a new token. In one-click deployments you can set it yourself with the `DIDACTA_SETUP_TOKEN` variable (at least 16 characters, url-safe alphabet).

## The steps

1. **Welcome** — an introduction to the wizard.
2. **Organization** — the organization name (required), the **academy language** (Spanish, English or Brazilian Portuguese; the wizard proposes your browser's; see [Languages](../configuration/languages.md)) and the public domain, pre-filled with the host you are connecting from. Under "Advanced settings" you can adjust the *slug* (auto-generated, DNS-safe).
3. **Modules** — the list of available modules. **Core** modules (courses, learning, assessments…) come pre-selected and cannot be unchecked; optional ones (community, gamification, Fundae, virtual classroom…) are up to you. Everything can be changed later under **Administration → Settings → Modules**.
4. **Your account** — name, email and password of the first administrator (**at least 12 characters**, with a strength meter).
5. **Done** — a quick tour (Courses, Community, Administration) and a recommendation to enable MFA. "Get started" opens the academy's welcome wizard (`/welcome`): name, logo and color, first course, students, payments and email.

!!! note "Two blocking wizards on first access"
    The administrator meets two: the **profile** one (`/onboarding`: photo, name, language and notices) and the **academy** one (`/welcome`). In **full mode** the academy one comes first (the photo is mandatory and is not demanded before letting them touch their academy); in [simple mode](../configuration/simple-mode.md) the profile one comes first, with the photo optional, and `/welcome` is the **3-question sign-up**. Anyone who does not administer only sees the profile one.

## Exactly what it creates

All in a single transaction:

- The 6 **system roles**: `super_admin`, `tenant_admin`, `formador`, `alumno`, `auditor`, `empresa_manager`.
- The **tenant** (organization) with its **verified primary domain** — the hostname you used to connect. If it is not `localhost`, `localhost` is also added as a secondary domain (handy for local administration).
- The **administrator user** with the `super_admin` role and a password hashed with argon2.
- Activation of the **modules** you chose (core modules always).
- A `setup.initialized` **audit log** entry.
- An **active session**: when you finish you are already signed in, with no second login.

!!! tip "The domain matters"
    Didacta resolves the tenant from the request **host**. Open the wizard from your installation's final domain (not from a temporary IP address) so it is registered as the primary domain. You can always add domains later under **Administration → Tenants**.

## After the wizard

1. **Administration → Branding** — logo, colors and sign-in screen copy.
2. **Administration → Settings → Notifications** — your real mail server (it points to Mailpit by default).
3. **Administration → AI providers** — the AI provider and key, if you are going to use the AI tutor, AI grading or AI content generation.
4. **Administration → Settings → Modules** — adjust which modules are active.

→ Follow the [visual walkthrough](visual-walkthrough.md) to see these steps, and the whole path up to the first certificate issued, screenshot by screenshot.
