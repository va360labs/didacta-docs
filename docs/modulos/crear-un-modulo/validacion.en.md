# Validation and tests

Before a module can be considered finished, it has to pass the contract validators and ship with tests.

## module-doctor

```bash
pnpm tsx scripts/module-doctor.ts modules/mi-modulo
pnpm tsx scripts/module-doctor.ts --all
```

It validates the declarative contract:

- `module.json` is valid JSON with the required fields (`name`, `version`, `edition`, `coreVersionRequired`, `tablePrefix`, `apiNamespace`).
- `edition` is `community` (modules are never enterprise). Exception: **marketplace-style** `module.json` files (those carrying `vendor`/`isolation`/`http`/`didacta`, validated by the host's strict schema) **do not carry** `edition` — the doctor exempts them and flags the legacy `edition`/`category`/`requiredLicenseFeature` keys as errors, since they would break installation with `MANIFEST_SCHEMA_INVALID`.
- `tablePrefix` follows the `mod_<slug-in-snake-case>_` convention and `apiNamespace` is `/modules/<slug>`.
- A `README.md` exists with the **9 mandatory sections**.
- If the module carries its own Prisma schema, every `@@map` uses the prefix.

## The module README

The 9 mandatory sections (the canonical example is `modules/access-groups/README.md`; the section headings are written in Spanish, which is what the doctor checks for):

```markdown
# mod.mi-modulo
## Edición
## Estado
## Resumen funcional
## Modelo de datos
## API pública
## Eventos
## Configuración
## Dependencias
```

## ee-fence

```bash
pnpm tsx scripts/ee-fence.ts
```

It guarantees the open-core model: no `.ee.*` file may live under `modules/*` (every module is Community), and the core's EE files carry the correct license header. It runs in CI and blocks the PR.

## Tests

- **Unit tests** in `modules/<slug>/tests/` with Vitest — the package's pure logic is tested without infrastructure. This includes a `contract.test.ts` that registers the module in a real `ModuleRegistry` (validating manifest + lifecycle).
- **Minimum 70% coverage** on services and handlers (a project rule).
- **Integration tests** for the host (the ephemeral database from `docker-compose.test.yml`), and **Playwright E2E tests** for the user flow you deliver: if there is no spec for your feature, write one.

```bash
pnpm --filter @didacta/mod-mi-modulo test   # the package's unit tests
bash scripts/test-local.sh                  # the full authorised suite
```

See [Running tests locally](../../comunidad/tests-locales.md).

## Final checklist

- [ ] `pnpm lint` and `pnpm typecheck` green.
- [ ] `pnpm tsx scripts/ee-fence.ts` with no errors.
- [ ] `pnpm tsx scripts/module-doctor.ts modules/mi-modulo` with no errors.
- [ ] `module.json` and `src/manifest.ts` consistent with each other.
- [ ] Package unit tests + host integration tests + E2E for the flow.
- [ ] A README with the 9 sections.
- [ ] A versioned migration applicable with `migrate deploy` (never `db push`).
