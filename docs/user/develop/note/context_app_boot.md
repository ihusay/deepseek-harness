# How `boot()` starts the Host

English | [中文](context_app_boot.zh.md)

## Summary

Follow `boot()` from a profile's configuration files to a running Cordis plugin tree. The CLI has already selected the profile and collected its patches when it calls this function. [CLI startup and profile resolution](cli_startup.md#profile)

## Table of contents

- [Startup order](#startup-order)
- [Startup failure and cleanup](#startup-failure)

<a id="startup-order"></a>
## Startup order

1. **Create the Host context.** `new Context()` creates the root Cordis context and its Fiber. `boot()` sets `ctx.baseUrl` to the directory containing the root `cordis.yml`, provides `dshHomePath`, and waits for `ctx.plugin(Loader)` to install the Loader. [`boot()` implementation](../../../../packages/boot/app-boot/src/index.ts#L1220)
2. **Run `prepare(ctx)`.** The optional callback runs after the Loader is available and before it mounts any entry from the profile. For `pnpm dsh web`, the callback from `runProfile()` supplies `profileContext` and the launch environment, installs the package resolver, and provides `cmdlineArgs`. The Web startup plugin can then inject `cmdlineArgs` to read options such as `--port`. [CLI callback](../../../../apps/cli/src/profile-boot.ts#L336) · [Web startup plugin](../../../../packages/bundle/web-app/src/startup.ts#L91)
3. **Mount the plugin tree.** `mountRootInclude()` registers the built-in `cordis:include` entry with the Loader. The Include reads the root `cordis.yml`, applies the ordered patches, and passes the resulting tree to the Loader. Plugin instances activate when their required services become available. [Root Include setup](../../../../packages/boot/app-boot/src/index.ts#L758) · [Patch application](../../../../vendor/include/src/index.ts#L449)
4. **Wait and audit.** `boot()` waits for the Loader's current work to settle, then calls `auditStartupEntries()`. If the root Include or a required entry fails to activate, the audit throws `StartupError`. Other inactive entries can produce a warning while successful plugins stay active. After a successful audit, `boot()` returns the root `ctx`. If the app already shut down during startup, `boot()` returns the context once it sees that the Loader has been disposed. [Startup audit](../../../../packages/boot/app-boot/src/index.ts#L1173)

<a id="startup-failure"></a>
## Startup failure and cleanup

If startup throws, `boot()` awaits `ctx.fiber.dispose()` to clean up the partially started Host before passing the error back to the CLI. A separate `diagnostics` context shares the Host logger and collects startup warnings and errors until cleanup finishes. For a `StartupError`, `boot()` attaches those messages to the error. The `finally` block disposes the diagnostics Fiber on both success and failure. [Failure handling](../../../../packages/boot/app-boot/src/index.ts#L1257)
