# Troubleshooting

Common installation problems and how to fix them.

| Symptom | Cause and fix |
| --- | --- |
| The app does not start and the log says `type "vector" does not exist` | Your Postgres does not have pgvector. Use the `pgvector/pgvector:pg16` image or install the extension (`CREATE EXTENSION vector;`). |
| `Error: P3009 — migrate found failed migrations` | A migration was left half-applied (for example because of an outage). Check the log, fix the cause and mark the migration with `prisma migrate resolve`; startup never deletes data on its own. |
| The browser loads but the API returns 502/timeout | The first start takes up to ~60-90 s (healthcheck with `start_period`). Check `docker compose -f docker-compose.alpha.yml logs -f didacta`. |
| No emails arrive | In the default installation, emails go to the local Mailpit instance (`http://localhost:8025`), not to the internet. Configure a real SMTP server under **Settings → Notifications**. |
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

- **Bugs and feedback**: open an issue on [GitHub](https://github.com/va360labs/didacta-io/issues) — there are bug, feedback and feature request templates. Before opening one, read [Reporting a bug](../comunidad/reportar-un-error.md): what data is needed and what to anonymise.
- **Security vulnerabilities**: follow the policy in [SECURITY.md](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md) — never publish them in an issue.
