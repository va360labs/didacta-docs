# Contribuir a Didacta

Didacta está en **alpha** camino de `v1.0.0`. Las contribuciones externas son bienvenidas — issues, feedback de instalación y PRs acotados — con la cautela de que la API y el schema aún pueden cambiar entre versiones alpha.

La guía canónica es [CONTRIBUTING.md](https://github.com/va360labs/didacta-io/blob/main/CONTRIBUTING.md); esto es el resumen operativo.

## Tu primera contribución

1. **CLA.** Cualquier PR queda bloqueado hasta firmar el Contributor License Agreement vía cla-assistant.io. El bot te lo pide automáticamente la primera vez; una sola firma vale para siempre.
2. **Issue antes de PR.** Para cambios no triviales, abre primero una issue con el problema o la propuesta. Para fixes pequeños (typo, doc) puedes ir directo al PR.
3. **Rama.** `git checkout -b <tipo>/<descripción-corta>` con tipos `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`, `test/`, `perf/`.
4. **Conventional Commits obligatorios**, con el cuerpo explicando el *por qué*. Sin atribución a IA en los commits (política del proyecto).
5. **Tests.** Todo PR que toque lógica de negocio incluye tests; coverage mínimo 70% en services.

## Validaciones antes de push

```bash
pnpm lint
pnpm typecheck
pnpm test
pnpm tsx scripts/ee-fence.ts        # fence open-core: .ee solo en el core
pnpm tsx scripts/module-doctor.ts   # contrato de módulos
```

La CI las ejecuta otra vez; si fallan, el PR no se mergea. Para las suites de integración con Docker, ver [Tests en local](tests-locales.md).

## Reglas duras (PR rechazado si se rompen)

- **Convención `.ee`**: archivos `*.ee.ts` y carpetas `ee/` solo dentro del core. Ningún módulo puede tener ficheros `.ee` — todos los módulos son Community.
- **Contrato de módulo**: tablas `mod_<slug>_*` con `tenant_id` + RLS, cero FKs cross-module, cero imports de código privado de otro módulo, `module.json` válido, lifecycle hooks implementados. Detalle en [Crear un módulo](../modulos/crear-un-modulo/index.md).
- **Sin dependencias copyleft**: MIT/Apache-2.0/BSD/ISC ✅ · LGPL caso a caso · GPL/AGPL/MPL/SSPL ❌. La CI corre `scripts/license-check.ts`.

## Estilo de código

- TypeScript estricto; no `any` salvo casos justificados con comentario.
- Prettier + ESLint (la CI lo verifica).
- Identificadores en inglés; comentarios y commits en español o inglés.
- Sin `console.log` en producción — logger Pino.

## Decisiones de arquitectura

Cambios que afecten al contrato de módulo, al modelo de licencias, al SDK o a APIs públicas requieren una **ADR** previa: proponla en una issue o GitHub Discussion con contexto, opciones y trade-offs.

## Contacto

- 💬 Preguntas técnicas: GitHub Discussions o Discord `#didacta-alpha` (durante alpha).
- 🐛 Bugs: issue con la plantilla de bug — cómo prepararla en [Reportar un error](reportar-un-error.md).
- 🔒 Seguridad: `security@didacta.io` — **nunca** un issue público. Ver [SECURITY.md](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md).
- 📜 Licensing / comercial: `licensing@didacta.io`.
