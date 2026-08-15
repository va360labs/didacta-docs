---
render_macros: true
---

# Contributing to Didacta

Didacta is in **{{ didacta_channel_en|lower }}** on its way to `v1.0.0`. External contributions are welcome — issues, installation feedback and focused PRs — with the caveat that the API and the schema can still change between pre-1.0 versions.

The canonical guide is [CONTRIBUTING.md](https://github.com/va360labs/didacta-io/blob/main/CONTRIBUTING.md); this is the operational summary.

## Your first contribution

1. **CLA.** Every PR is blocked until you sign the Contributor License Agreement through cla-assistant.io. The bot asks you automatically the first time; one signature is good forever.
2. **Issue before PR.** For non-trivial changes, open an issue first with the problem or the proposal. For small fixes (a typo, a doc) you can go straight to a PR.
3. **Branch.** `git checkout -b <type>/<short-description>` with the types `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`, `test/`, `perf/`.
4. **Conventional Commits are mandatory**, with the body explaining the *why*. No AI attribution in commits (project policy).
5. **Tests.** Every PR that touches business logic includes tests; minimum 70% coverage on services.

## Checks before you push

```bash
pnpm lint
pnpm typecheck
pnpm test
pnpm tsx scripts/ee-fence.ts        # open-core fence: .ee only in the core
pnpm tsx scripts/module-doctor.ts   # the module contract
```

CI runs them again; if they fail, the PR is not merged. For the Docker-based integration suites, see [Running tests locally](tests-locales.md).

## Hard rules (a PR is rejected if they are broken)

- **The `.ee` convention**: `*.ee.ts` files and `ee/` folders only inside the core. No module may contain `.ee` files — every module is Community.
- **The module contract**: `mod_<slug>_*` tables with `tenant_id` + RLS, zero cross-module foreign keys, zero imports of another module's private code, a valid `module.json`, lifecycle hooks implemented. Details in [Building a module](../modulos/crear-un-modulo/index.md).
- **No copyleft dependencies**: MIT/Apache-2.0/BSD/ISC ✅ · LGPL case by case · GPL/AGPL/MPL/SSPL ❌. CI runs `scripts/license-check.ts`.

## Code style

- Strict TypeScript; no `any` except in justified cases, with a comment.
- Prettier + ESLint (CI verifies it).
- Identifiers in English; comments and commits in Spanish or English.
- No `console.log` in production — use the Pino logger.

## Architecture decisions

Changes affecting the module contract, the licensing model, the SDK or public APIs require an **ADR** first: propose it in an issue or a GitHub Discussion with context, options and trade-offs.

## Contact

- 💬 Technical questions: GitHub Discussions or Discord `#didacta-alpha` (pre-1.0 phase).
- 🐛 Bugs: an issue using the bug template — how to prepare it in [Reporting a bug](reportar-un-error.md).
- 🔒 Security: `security@didacta.io` — **never** a public issue. See [SECURITY.md](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md).
- 📜 Licensing / commercial: `licensing@didacta.io`.
