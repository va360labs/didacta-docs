# mod.hello-world — Reference module

<span class="didacta-chip didacta-chip--community">Community</span> · **Example** category (can be disabled)

## What it does

An example module and a **starting template** for building new modules. It demonstrates the four elements of the contract:

1. A valid **manifest** parsed with the core-kernel's `parseModuleManifest`.
2. The four **lifecycle hooks** (`onRegister`, `onEnable`, `onDisable`, `onUninstall`), with comments describing what a real module would do in each one.
3. A **domain service** (`HelloWorldService`) that receives the `ModuleContext` and uses `eventBus` and `i18n`.
4. A **contract test suite** that registers the module in a real `ModuleRegistry` and verifies it end to end.

## How to use it

The procedure for starting a new module: copy the folder, rename `name`/`tablePrefix`/`apiNamespace`/permissions, write the domain logic, implement the lifecycle and update the tests — the full guide is in [Building a module](../crear-un-modulo/index.md).

Its README lists the four anti-patterns every module honours: never import another module's code, never touch another module's tables with Prisma, never modify the core for your own features, and never emit events without declaring them.

## Data, API and events

- It declares a `tablePrefix` but **creates no tables** (it does not need any).
- It declares the `apiNamespace /modules/hello-world` but exposes no real endpoints.
- It declares the `hello-world.greeting.requested` event and the `hello-world.greeting.read` permission as an example of the contract.

## Configuration

None.
