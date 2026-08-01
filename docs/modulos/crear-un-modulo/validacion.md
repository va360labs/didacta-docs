# Validación y tests

Antes de considerar un módulo terminado, tiene que pasar los validadores del contrato y traer tests.

## module-doctor

```bash
pnpm tsx scripts/module-doctor.ts modules/mi-modulo
pnpm tsx scripts/module-doctor.ts --all
```

Valida el contrato declarativo:

- `module.json` es JSON válido con los campos obligatorios (`name`, `version`, `edition`, `coreVersionRequired`, `tablePrefix`, `apiNamespace`).
- `edition` es `community` (los módulos nunca son enterprise).
- `tablePrefix` sigue la convención `mod_<slug-en-snake>_` y `apiNamespace` es `/modules/<slug>`.
- Existe `README.md` con las **9 secciones obligatorias**.
- Si el módulo lleva schema Prisma propio, todas las `@@map` usan el prefijo.

## El README del módulo

Las 9 secciones obligatorias (el ejemplar canónico es `modules/access-groups/README.md`):

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

Garantiza el modelo open-core: ningún fichero `.ee.*` puede vivir en `modules/*` (todos los módulos son Community), y los ficheros EE del core llevan la cabecera de licencia correcta. Corre en CI y bloquea el PR.

## Tests

- **Unit** en `modules/<slug>/tests/` con Vitest — la lógica pura del paquete se testea sin infraestructura. Incluye un `contract.test.ts` que registra el módulo en un `ModuleRegistry` real (valida manifest + lifecycle).
- **Coverage mínimo 70%** en services y handlers (regla del proyecto).
- **Integración** para el host (BD efímera de `docker-compose.test.yml`), y **E2E de Playwright** para el flujo de usuario entregado: si no existe spec de tu funcionalidad, créala.

```bash
pnpm --filter @didacta/mod-mi-modulo test   # unit del paquete
bash scripts/test-local.sh                  # suite completa autorizada
```

Ver [Tests en local](../../comunidad/tests-locales.md).

## Checklist final

- [ ] `pnpm lint` y `pnpm typecheck` en verde.
- [ ] `pnpm tsx scripts/ee-fence.ts` sin errores.
- [ ] `pnpm tsx scripts/module-doctor.ts modules/mi-modulo` sin errores.
- [ ] `module.json` y `src/manifest.ts` coherentes entre sí.
- [ ] Tests unit del paquete + integración del host + E2E del flujo.
- [ ] README con las 9 secciones.
- [ ] Migración versionada aplicable con `migrate deploy` (nunca `db push`).
