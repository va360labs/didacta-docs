# Third-party modules (signed ZIP)

Beyond the bundled modules, Didacta can run modules **packaged as a signed ZIP** that a `super_admin` installs from **Administration → Module marketplace**. This is the intended route for the future module marketplace.

!!! note "Status: functional, still evolving"
    The installation pipeline, the sandbox and the routing are implemented and operational. The **public marketplace catalog does not exist yet** (today installation happens by uploading the ZIP by hand), modules from the `community` vendor are not accepted yet (only packages signed by Didacta, or a deliberate direct upload), and uninstalling neither reverts migrations nor drops tables.

## The package format

```
<module>.zip                          (max. 50 MB)
├── manifest.jwt                      # signed manifest (JWS compact, ES256)
├── package.json                      # { name, version, main: "dist/index.js" }
├── dist/index.js                     # CommonJS bundle of the backend
├── dist/ui/<surface>.js              # UI bundles per surface (admin, student…)
└── prisma/migrations/<ts>_<x>.sql    # plain SQL migrations
```

- `manifest.jwt` is an **ES256** JWT whose payload is the module's full manifest (name, version, tables, permissions, events, UI surfaces, network/DB limits…).
- The signature is verified against Didacta's public keys; a **direct unsigned upload** is also accepted for your own packages, and is explicitly flagged as such.

## What installation validates

1. Size, ZIP structure and path safety (no `..`, no absolute paths).
2. Manifest signature and internal consistency: `tablePrefix` and `apiNamespace` must derive from the name (`mod.my-module` → `mod_my_module_` and `/modules/my-module`).
3. A non-reserved name (it cannot impersonate a built-in module) and compatibility with the core version.
4. **Linting of the SQL migrations**: they may only touch tables carrying their prefix; no `CREATE FUNCTION`, `GRANT`, `TRUNCATE`, `COPY`, roles or schemas.
5. **Static linting of the bundle**: no access to `fs`, no direct networking, no processes, no `eval`; only a closed set of allowed imports (`@didacta/core-kernel`, `zod`, cryptography and pure utilities).

## The sandbox at runtime

Third-party code runs in an **isolated Node VM** with scoped resources injected by the host:

| Resource | What it allows |
| --- | --- |
| `ctx.db` | SQL **against its own tables only** (`mod_<slug>_*`), with prepared statements, a 10 s maximum timeout, a 10,000-row cap and no DDL at runtime. |
| `ctx.http` | Outbound calls only to the hosts declared in the manifest, with rate limiting and SSRF protection. |
| `ctx.didacta` | The core's public API, with a permission matrix and idempotency. |
| `ctx.secrets` | An AES-256-GCM encrypted store, isolated per tenant and per module. |
| `ctx.jobs` | Periodic jobs (`onJobTick`) on top of BullMQ. |

Its HTTP routes are published under `/api/v1/modules/<slug>/…` and its UI is served as per-surface bundles (`admin`, `alumno`…).

!!! warning "Third-party UI is not sandboxed"
    UI bundles run in the same browser context as the Didacta web app (there is no UI sandbox yet). Only install packages you trust.

## Signing your own packages

Official packaging and signing is done with VA360 LABS tooling (`scripts/marketplace/sign-package.ts`, against the signing issuer). For modules internal to your organization you can use the unsigned direct upload, accepting the provenance warning.

## Reference

The complete real-world example of a third-party module is `mod.migrator-learndash` (it lives in the repository and is packaged in this format). The package contract is in `packages/module-package-spec`.
