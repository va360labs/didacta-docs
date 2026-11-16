# Troubleshooting

Common installation problems and how to fix them.

| Symptom | Cause and fix |
| --- | --- |
| The app does not start and the log says `type "vector" does not exist` | Your Postgres does not have pgvector. Use the `pgvector/pgvector:pg16` image or install the extension (`CREATE EXTENSION vector;`). |
| `Error: P3009 — migrate found failed migrations` | A migration was left half-applied (for example because of an outage). Check the log, fix the cause and mark the migration with `prisma migrate resolve`; startup never deletes data on its own. |
| The browser loads but the API returns 502/timeout | The first start takes up to ~60-90 s (healthcheck with `start_period`). Check `docker compose -f docker-compose.alpha.yml logs -f didacta`. |
| No emails arrive | In the default installation, emails go to the local Mailpit instance (`http://localhost:8025`), not to the internet. Configure a real SMTP server under **Administration → Settings → Notifications**. |
| The `/setup` wizard says the token is missing or invalid | The initial sign-up requires `/setup?token=…`. Check the logs: `docker compose -f docker-compose.alpha.yml logs didacta \| grep "Setup token"` (if the instance restarts before you finish, it prints a new one), or set `DIDACTA_SETUP_TOKEN` (16+ characters). |
| `/metrics` answers 401 after upgrading | In production `/metrics` requires `METRICS_TOKEN`. Define it and give it to Prometheus as `Authorization: Bearer …`. |
| The signed audit-log export answers 503 | `AUDIT_REPORT_HMAC_KEY` is missing (required with `NODE_ENV=production`). |
| The API does not start and mentions the encryption key | In production, if there is no `TENANT_SETTINGS_ENC_KEY` and the key file can be neither read nor created, it **does not start**. Check that `/app/data` is a **writable** volume or set the key yourself (`openssl rand -hex 32`). If the volume is mounted exactly at `STORAGE_ROOT`, define `SECRETS_DIR` on another volume. |
| An address or subdomain answers 404 | Didacta answers 404 to a host that belongs to no academy. Register the domain (**Administration → Tenants**) or, if you need to serve any host, `DIDACTA_ALLOW_UNKNOWN_HOSTS=true`. See [Branding → Own domain](../configuration/branding.md#own-domain). |
| An email arrives with links to `localhost` | `WEB_PUBLIC_URL` is missing and the academy has no verified primary domain. With compose, it has to be passed to the container (see [Passing more variables](docker-compose.md#passing-more-variables-to-the-container)). |
| A `.env` variable has no effect with compose | The `didacta` service only receives the variables its `environment` block declares; see [Passing more variables](docker-compose.md#passing-more-variables-to-the-container). |
| Many `429`s | Community per-minute limits (100 per academy with a session, 30 per IP without one) and the login's own limit (per IP and per account). See `RATE_LIMIT_*` in [Environment variables](../configuration/environment-variables.md). With a badly configured proxy (`TRUSTED_PROXY_HOPS`) every visitor shares one IP. |
| Port already in use | Change `WEB_PORT`, `API_PORT`, `POSTGRES_PORT`… in `.env`. |
| The app starts but I cannot sign in / the session drops | Check that `AUTH_SECRET` has **32+ characters** and has not changed between restarts (changing it invalidates every session). |

## Where to look

```bash
# Service status and healthchecks
docker compose -f docker-compose.alpha.yml ps

# Application logs (bootstrap, migrations, errors)
docker compose -f docker-compose.alpha.yml logs -f didacta

# API health
curl -fsS http://localhost:4000/healthz
```

## Getting help

- **Bugs and feedback**: open an issue on [GitHub](https://github.com/va360labs/didacta-io/issues) — there are bug, feedback and feature request templates. Before opening one, read [Reporting a bug](../community/reporting-a-bug.md): what data is needed and what to anonymise.
- **Security vulnerabilities**: follow the policy in [SECURITY.md](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md) — never publish them in an issue.
