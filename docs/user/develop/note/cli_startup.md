# How `pnpm dsh web` starts

English | [中文](cli_startup.zh.md)

## Summary

Follow `pnpm dsh web` from command-line parsing to the first React render. To understand which plugins and settings your checkout will use, run this command from the repository root. It prints the composed profile and exits before mounting its plugins.

```sh
pnpm dsh web --dump-config
```

## Table of contents

- [Select and assemble the profile](#profile)
- [Start the Host and serve the page](#host)
- [Load and render the browser application](#browser)

<a id="profile"></a>
## Select and assemble the profile

1. The root [`dsh` script](../../../../package.json) runs `node --import tsx/esm apps/cli/src/bin.ts web`. [`parseDshArgs()`](../../../../apps/cli/src/args.ts) expands `web` to `--profile web`, and [`runCli()`](../../../../apps/cli/src/bin.ts) passes the selected profile and remaining arguments to `runProfile()`.
2. [`runProfile()`](../../../../apps/cli/src/profile-boot.ts) prepares the profile and package resolution, then calls `boot()` with the composed patches. The shipped Web profile layers [`dsh-base`](../../../../packages/bundle/base/cordis.patch.yml) and [`dsh-web-app`](../../../../packages/bundle/web-app/cordis.patch.yml). Local profile and command-line patches can change the resulting plugin tree. [Profile and patch reference](../../../../apps/cli/reference/README.md)
3. [`boot()`](../../../../packages/boot/app-boot/src/index.ts#L1220) creates the Host's root Cordis `Context` and installs the Loader. The CLI supplies launch information such as `cmdlineArgs` before the Loader mounts the profile's plugin tree.

### How the profile is resolved

1. `composeProfile()` calls `prepareProfile('web')`, which calls `loadProfile()` to locate `$DSH_HOME/profiles/web/`. If the directory has no `package.json`, `loadProfile()` initializes it from `PROFILE_TEMPLATES.web`. The manifest's `dsh.profile.bundles` lists `@deepseek-ai/dsh-base` first and `@deepseek-ai/dsh-web-app` second by default. [CLI profile preparation](../../../../apps/cli/src/profile-boot.ts) · [Profile loading](../../../../packages/boot/app-boot/src/profile.ts) · [Built-in templates](../../../../packages/util/official-bundles/src/bundles.ts)
2. `loadProfileDirectory()` walks that list in order. It looks for each bundle in the dsh installation first, then in the profile, and reads the YAML files named by the bundle's `dsh.bundle.patch` field. For example, the `dsh-base` patch uses `insert` to add entries such as `llm` and `session`, and the `dsh-web-app` patch changes selected base entries and adds Web entries such as `webserver`. `loadOverlayPatches()` reads the files, and `parsePatchList()` parses each as a YAML array. [Base bundle manifest](../../../../packages/bundle/base/package.json) · [Base patch](../../../../packages/bundle/base/cordis.patch.yml) · [Web patch](../../../../packages/bundle/web-app/cordis.patch.yml) · [YAML parsing](../../../../packages/boot/app-boot/src/index.ts)
3. `composeProfile()` builds the package resolution table and reads any files named by `--patch`. Before calling `boot()`, `runProfile()` gathers the patches through `readProfilePatches()`. The order is bundle patches, `$DSH_HOME/profiles/web/cordis.patch.yml`, `$DSH_HOME/cordis.patch.yml`, then any `--patch` files in command-line order. If the telemetry switch contributes a patch, that patch goes last. [Patch ordering](../../../../packages/boot/app-boot/src/profile-context.ts)
4. `prepareProfile()` writes an empty `cordis.yml` to anchor the profile. During boot, `mountRootInclude()` passes that file and the ordered patches to `cordis:include`. The Include applies the patches over the empty list to produce the entries that the Loader mounts. [Root Include setup](../../../../packages/boot/app-boot/src/index.ts) · [Patch application](../../../../vendor/include/src/index.ts)
5. Run `pnpm dsh web --dump-config` to inspect the composed profile before plugin mounting. To inspect the effect of an additional overlay, pass `--patch FILE` to the same command.

<a id="host"></a>
## Start the Host and serve the page

1. [`web-startup`](../../../../packages/bundle/web-app/src/startup.ts) reads `cmdlineArgs` and provides the resolved Web options as `webStartup`. The [Web bundle configuration](../../../../packages/bundle/web-app/cordis.patch.yml) uses those values to configure the HTTP listener and other Web plugins.
2. [`WebServer`](../../../../packages/host/webserver/src/index.ts) registers itself as `ctx.webServer` through `super(ctx, 'webServer')`. During initialization it calls `server.listen()`. The default bind address is `127.0.0.1:3080`.
3. The [`web-runtime` plugin](../../../../packages/bundle/web-app/src/index.ts) injects `webServer`, resolves the built frontend's `dist/index.html`, and loads `frontend-static` through `ctx.plugin()`. [`frontend-static`](../../../../packages/host/frontend-static/src/index.ts) waits for `webServer` and `connection`, then calls `ctx.effect()` to register the static-file fallback. The effect removes the fallback when the plugin unloads.
4. Once the required plugins are ready, `web-runtime` prints a URL containing the startup token and opens the default browser when enabled. [Web startup and browser handoff](../../../../apps/cli/reference/README.md#web-profile)

<a id="browser"></a>
## Load and render the browser application

1. The browser opens the tokenized URL. Authentication exchanges the token for a signed cookie and redirects to the clean URL. The fallback reads `dist/index.html` and runs `renderIndex()` to add the Host's startup data before returning the page. [Web authentication flow](../../../../packages/bundle/web-app/README.md) · [Static file handling](../../../../packages/host/frontend-static/src/index.ts)
2. The built page's module script runs the code from [`apps/web/src/main.ts`](../../../../apps/web/src/main.ts). It finds `#root`, creates `AppWebEntry`, and calls `entry.run()`. [HTML entry](../../../../apps/web/index.html)
3. [`AppWebEntry.run()`](../../../../packages/client/web/src/boot.ts) waits for the bootstrap data, creates a browser-side Cordis `Context`, activates client plugins, and hands the mount point to the [UI renderer](../../../../packages/client/ui-renderer/src/client/index.ts). The renderer calls React's `createRoot()` and `root.render()`.
