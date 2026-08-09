# Running the tests locally

Didacta's tests run **locally** through `scripts/test-local.sh`, the authorised runner to use before merging. GitHub Actions CI validates typecheck, lint, ee-fence, gitleaks and license-check; the unit and integration suites run on your machine.

## Requirements

- Docker Desktop (or a Docker daemon) running.
- `pnpm` on your PATH, at the version in the root `package.json`'s `packageManager` field.
- Ports **5433** and **6380** free (the ephemeral test Postgres and Redis).

## Commands

```bash
bash scripts/test-local.sh          # full: pre-flight + build + unit + integration
bash scripts/test-local.sh unit     # pre-flight + unit only
bash scripts/test-local.sh integ    # pre-flight + build + integration only
```

They are also available as pnpm scripts: `pnpm test:local`, `pnpm test:local:unit`, `pnpm test:local:integ`. On Windows, use Git Bash (or `scripts\test-local.cmd`, which delegates to it).

## What the runner does

1. **Pre-flight**: checks Docker, that the ports are free, and that no `didacta-*-test` containers are left over from previous runs.
2. Generates the Prisma client and builds the internal packages the tests import (`@didacta/database`, `@didacta/core-kernel`, `@didacta/core-registry`, `@didacta/license-sdk`).
3. Runs the **unit** suite (`pnpm test`, Vitest through turbo).
4. Brings up `docker-compose.test.yml` (postgres-test on 5433, redis-test on 6380 — the database on tmpfs, Redis without persistence), runs the **integration** suite and always tears the compose down afterwards, even if Vitest fails.

## Quick suites without Docker

- `pnpm typecheck` — typecheck across the whole monorepo.
- `pnpm test` — the unit tests only (they need no infrastructure).

## E2E (Playwright)

The E2E tests in `apps/e2e` need the API and the web app running (locally with `pnpm dev`, or the `docker-compose.alpha.yml` stack). See `apps/e2e/README.md` in the repository.
