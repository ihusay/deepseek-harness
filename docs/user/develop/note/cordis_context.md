# Cordis Context and its core services

English | [中文](cordis_context.zh.md)

## Summary

Use this note to trace an operation such as `ctx.plugin()`, `ctx.provide()`, or `ctx.on()` from a plugin's Context to the object that implements it. Learn which plugin entry points Cordis accepts and how the registry creates a Fiber for each mount.

## Table of contents

- [Core objects](#core-objects)
  - [Context](#context)
- [Plugin](#plugin)
  - [`Plugin.Runtime`](#plugin-runtime)
  - [`Plugin.inject`](#plugin-inject)
- [Registry](#registry)
  - [`ctx.plugin()`](#ctx-plugin)
  - [`ctx.inject()`](#ctx-inject)
- [Fiber](#fiber)
  - [Lifecycle and cleanup](#fiber-lifecycle)
    - [Stable state transitions](#fiber-stable-transitions)
    - [Construction](#fiber-construction)
    - [Dependency resolution](#fiber-dependency-resolution)
    - [Dependency loss](#fiber-dependency-loss)
- [ReflectService](#reflect-service)
- [EventsService](#events-service)
- [LoggerService](#logger-service)

<a id="core-objects"></a>
## Core objects

`new Context()` creates the root Fiber and the built-in services, then returns a Proxy. `ReflectService` installs method aliases on `ctx`, so `ctx.plugin()` delegates to the registry, `ctx.provide()` to reflection, and `ctx.on()` to the event service. [Context constructor](../../../../vendor/cordis/src/context.ts#L73) · [Method aliases](../../../../vendor/cordis/src/reflect.ts#L233)

| Object | Responsibility |
|---|---|
| `Context` | Gives plugins one entry point and a scope for resolving services. A child context inherits from its parent, and `ctx.isolate(name)` creates a child scope for resolving that service name independently. [Context](../../../../vendor/cordis/src/context.ts#L42) · [`isolate()`](../../../../vendor/cordis/src/context.ts#L126) |
| `ReflectService` · `ctx.reflect` | Registers service implementations through `provide()`. Its Proxy handler resolves property reads such as `ctx.webServer` using the caller's context and service scope. [Service registration](../../../../vendor/cordis/src/reflect.ts#L313) · [Property lookup](../../../../vendor/cordis/src/reflect.ts#L144) |
| `RegistryService` · `ctx.registry` | Resolves a plugin to its entry point, tracks plugin runtime records, and creates a Fiber for each `ctx.plugin()` registration. [Plugin registration](../../../../vendor/cordis/src/registry.ts#L331) |
| `Fiber` · `ctx.fiber` | Manages one plugin instance. It tracks dependencies and configuration, activates the plugin, and runs the cleanup functions registered through `ctx.effect()` when the plugin unloads. [Fiber](../../../../vendor/cordis/src/fiber.ts#L214) · [`effect()`](../../../../vendor/cordis/src/fiber.ts#L443) |
| `EventsService` · `ctx.events` | Registers listeners through `ctx.on()` and dispatches them through modes such as `emit`, `parallel`, and `waterfall`. A listener's registration belongs to its plugin's Fiber. [EventsService](../../../../vendor/cordis/src/events.ts#L130) · [Listener registration](../../../../vendor/cordis/src/events.ts#L298) |
| `LoggerService` · `ctx.logger` | Sends log messages to registered exporters. During Host startup, `boot()` adds an exporter to collect warnings and errors for diagnostics. [LoggerService](../../../../vendor/cordis/src/logger.ts#L194) · [Startup exporter](../../../../packages/boot/app-boot/src/index.ts#L1280) |

<a id="context"></a>
### Context

`Context` gives plugin code access to its scope and built-in services. [Source](../../../../vendor/cordis/src/context.ts#L42)

#### Core attributes

| Property | Purpose |
|---|---|
| `fiber` | The Fiber that owns effects registered through this context. The root context points to the root Fiber. A plugin's context points to the Fiber created for that mount. |
| `runtime` | Alias for `ctx.fiber.runtime`, the shared record for the plugin entry point. It is `null` on the root Fiber. |
| `baseUrl` | Optional base URL. The Loader can use it to resolve relative plugin module specifiers. |
| `reflect` / `registry` | `reflect` resolves and registers services. `registry` resolves plugin entry points and creates Fibers for mounts. |
| `events` / `logger` | `events` registers and dispatches listeners. `logger` can write logs directly through `ctx.logger.info(...)` or create a named logger through `ctx.logger(name)`. |
| `ctx[Context.root]` | The same root Context for the entire tree. `ctx.fiber.parent` gives the context that mounted the current plugin. |
| `ctx[Context.isolate]` / `ctx[Context.intercept]` | Maps service names to scope labels and intercept metadata, respectively. `isolate()` and `intercept()` create child contexts carrying the corresponding entries. |
| `ctx[Context.filter]` / `disposer[Context.effect]` | The optional event filter selects listeners when this context is passed as the dispatch receiver. `Context.effect` is the symbol key for effect diagnostics on a disposer. |

#### Core methods

The `Context` class defines the methods for creating child contexts and checking the Context type. [Class methods](../../../../vendor/cordis/src/context.ts#L54)

| Method | Purpose |
|---|---|
| [`new Context()`](../../../../vendor/cordis/src/context.ts#L73) | Creates the root Fiber and built-in services, then returns the root Context proxy. |
| [`extend(meta?)`](../../../../vendor/cordis/src/context.ts#L101) | Creates a child context that inherits from the current context. Own properties of `meta`, including symbol keys, take precedence on the child. The child inherits the current Fiber unless `meta` supplies one. |
| [`isolate(name, label?)`](../../../../vendor/cordis/src/context.ts#L126) | Creates a child context where reads and registrations for `name` use the given label. It creates a fresh symbol by default. Reuse a label to share a scope. A direct read under an isolated plugin scope throws if the required service is unavailable there. |
| [`intercept(name, config)`](../../../../vendor/cordis/src/context.ts#L147) | Creates a child context carrying configuration for a named service. The service combines metadata from the dependency declaration and enclosing contexts when it resolves its intercept configuration. |
| [`Context.is(value)`](../../../../vendor/cordis/src/context.ts#L63) | Tests whether a value is a Cordis Context and narrows its TypeScript type. The global symbol marker works across copies of Cordis. |

`ReflectService` forwards the methods below to the Fiber or built-in service for the current context. The declaration `interface Context extends Pick<Fiber, 'effect'>` gives `ctx.effect` the signature of `Fiber.effect`. At runtime, `mixin('fiber', ['runtime', 'effect'])` installs the forwarding. [Type declaration](../../../../vendor/cordis/src/fiber.ts#L9) · [Forwarding registration](../../../../vendor/cordis/src/reflect.ts#L255)

| Method | Purpose |
|---|---|
| [`effect(setup, label?)`](../../../../vendor/cordis/src/fiber.ts#L446) | Calls `ctx.fiber.effect()`. Setup runs immediately. The returned disposer can clean up the effect before the owning Fiber unloads. |
| [`plugin(plugin, config?)`](../../../../vendor/cordis/src/registry.ts#L331) | Mounts a plugin under the current Fiber and returns an awaitable child Fiber handle. |
| [`inject(deps, callback)`](../../../../vendor/cordis/src/registry.ts#L315) | Mounts a child Fiber that runs the callback with its own context once the named services are available. |
| [`get(name, strict?)`](../../../../vendor/cordis/src/reflect.ts#L269) / [`set(name, value)`](../../../../vendor/cordis/src/reflect.ts#L290) | `get()` looks up a service in the current scope without requiring an `inject` declaration and returns `undefined` when no implementation matches. By default it only returns active providers' implementations. `set()` replaces the value of a service already registered by the current Fiber and throws if that registration is missing or belongs to a different Fiber. |
| [`provide(name, value?, check?)`](../../../../vendor/cordis/src/reflect.ts#L313) | Registers a service under the current scope label and returns its disposer. An optional `check` determines whether the implementation is available to dependents. |
| [`accessor(name, options)`](../../../../vendor/cordis/src/reflect.ts#L385) / [`mixin(source, mixins)`](../../../../vendor/cordis/src/reflect.ts#L404) | Define a computed context property or expose selected members of another context property directly on `ctx`. Both return disposers. |
| [`on(name, listener)`](../../../../vendor/cordis/src/events.ts#L298) / [`once(name, listener)`](../../../../vendor/cordis/src/events.ts#L321) | Register recurring or one-shot listeners and return unsubscribe functions. Registrations belong to the current Fiber. |
| [`emit()`](../../../../vendor/cordis/src/events.ts#L207) / [`parallel()`](../../../../vendor/cordis/src/events.ts#L193) / [`bail()`](../../../../vendor/cordis/src/events.ts#L232) / [`serial()`](../../../../vendor/cordis/src/events.ts#L218) / [`waterfall()`](../../../../vendor/cordis/src/events.ts#L250) | Dispatch events using the corresponding `EventsService` mode. |

For example, `const child = ctx.isolate('cache')` creates an independent scope for `cache` and inherits `ctx.fiber`. Calling `child.plugin(...)` creates a new Fiber, and the new plugin looks for `cache` in the child's scope. [Scope creation](../../../../vendor/cordis/src/context.ts#L126) · [Fiber creation](../../../../vendor/cordis/src/registry.ts#L345)

<a id="plugin"></a>
## Plugin

`Plugin<T>` defines the three entry points `ctx.plugin()` accepts. Each entry point receives the plugin's Context and configuration. `T` is the configuration type passed to the entry point. [Plugin type definition](../../../../vendor/cordis/src/registry.ts#L107)

| Type | How it starts |
|---|---|
| `Plugin.Function<T>` | Calls the function directly with `plugin(ctx, config)`. |
| `Plugin.Constructor<T>` | Creates an instance with `new PluginClass(ctx, config)`. |
| `Plugin.Object<T>` | Executes the object's `apply(ctx, config)` entry point. |

All three entry points share the optional fields defined by `Plugin.Base<T>`. [Plugin metadata](../../../../vendor/cordis/src/registry.ts#L115)

| Field | Type | Purpose |
|---|---|---|
| `name` | `string` | Names the plugin in diagnostics. Functions and classes usually use the `.name` supplied by JavaScript, and object plugins can set it explicitly. |
| `Config` | `StandardSchemaV1<any, T>` | Defines how to validate and transform configuration. The plugin entry point receives the result from the schema. |
| `inject` | `Inject` | Declares the service names required for startup. The Fiber looks for available services by those names in its scope. An object can also supply intercept configuration for each service. [Dependency checks](../../../../vendor/cordis/src/fiber.ts#L625) |
| `provide` | `string` or `string[]` | **The plugins in this repository currently leave this field unset.** An ordinary plugin's entry point can call `ctx.provide(name, value)` to register a service. [Plugin mounting](../../../../vendor/loader/src/config/entry.ts#L274) · [Service registration](../../../../vendor/cordis/src/reflect.ts#L313) |
| `intercept` | `Dict<boolean>` | Declares which services' intercept configuration the plugin reads. **The plugins in this repository currently leave this field unset.** [Field definition](../../../../vendor/cordis/src/registry.ts#L126) |

Uppercase `Config` is the schema on the plugin definition. The registry stores it in the shared `Runtime`. Lowercase `config` is the input to `ctx.plugin(plugin, config)` for one mount, and that mount's Fiber stores it. The Fiber resolves the configuration before it calls the plugin entry point. [Registry registration](../../../../vendor/cordis/src/registry.ts#L331) · [Fiber startup](../../../../vendor/cordis/src/fiber.ts#L273)

The Loader can mount a module that exports `apply` and `inject` as an object plugin. A module can declare both plugin-wide dependencies and dependencies for individual features. See the [`tool-fs` example under `ctx.inject()`](#ctx-inject).

<a id="plugin-runtime"></a>
### `Plugin.Runtime`

`Plugin.Runtime` is the record shared by all mounts of a plugin entry point. The registry stores these records in a `Map<Function, Plugin.Runtime>` keyed by the entry point's function reference. Two object plugins that use the same `apply` function also share a record. [Registry definition](../../../../vendor/cordis/src/registry.ts#L212) · [Entry point resolution](../../../../vendor/cordis/src/registry.ts#L237)

| Field | Purpose |
|---|---|
| `name` | Comes from the plugin definition at first registration and appears in diagnostics. |
| `callback` | Holds the shared executable entry point and serves as the registry lookup key. |
| `fibers` | Tracks the entry point's current mounts, including those waiting for dependencies. |
| `Config` | Holds the optional schema from the first registration, which each Fiber uses to resolve its own configuration. |

Each `ctx.plugin()` call creates a new Fiber with its own raw configuration, Context, and lifecycle state. Unloading an individual Fiber removes it from `fibers`. When the last Fiber is removed, the registry deletes the `Runtime` record. [Runtime definition](../../../../vendor/cordis/src/registry.ts#L151) · [Fiber creation](../../../../vendor/cordis/src/registry.ts#L345) · [Fiber removal](../../../../vendor/cordis/src/fiber.ts#L292)

<a id="plugin-inject"></a>
### `Plugin.inject`

| Declaration site | Syntax | Which Fiber waits |
|---|---|---|
| Module plugin | Export `export const inject = ['fs']` alongside `apply`. | The plugin Fiber. [Example](../../../../packages/fs/tool-fs/src/index.ts#L24) |
| Object, class, or function plugin | Use `{ inject: ['fs'], apply }` on an object, `static inject = ['fs']` on a class, or set `apply.inject = ['fs']` on a function. | The plugin Fiber. [Mount implementation](../../../../vendor/cordis/src/registry.ts#L331) |
| Class decorator | Add `@Inject('loader')` to a class. You can stack decorators. | The plugin Fiber. The decorator adds the dependency to the class's `static inject`. [Example](../../../../packages/boot/hmr/src/index.ts#L211) |
| Method decorator | Add `@Inject('timer')` to a class method. You can stack decorators. | The method's child Fiber. Cordis calls the method once its declared dependencies are available. [Implementation](../../../../vendor/cordis/src/registry.ts#L54) |
| Loader YAML entry | Add `inject: [fs]` to the entry. | The Fiber for that mount. The Loader merges the declaration into `fiber.inject` in its `internal/plugin` listener before the Fiber checks dependencies. The YAML declaration takes precedence for matching service names. [Merge call](../../../../vendor/loader/src/index.ts#L154) · [Merge rules](../../../../vendor/cordis/src/registry.ts#L86) |

<a id="registry"></a>
## Registry

The registry manages plugin mount records. Call these methods through `ctx.registry`. The Context also exposes `ctx.plugin()` and `ctx.inject()` directly. [RegistryService source](../../../../vendor/cordis/src/registry.ts#L210)

### Core attributes

| Property | Purpose |
|---|---|
| `size` | The current number of `Runtime` records. |
| `counter` | Allocates the next Fiber identifier on each read. |

### Core methods

| Method | Purpose |
| --- | --- |
| [`new RegistryService(ctx)`](../../../../vendor/cordis/src/registry.ts#L214) | Binds the registry to a Context. The registry starts with an empty runtime table and an identifier counter at zero. |
| [`plugin(plugin, config?)`](../../../../vendor/cordis/src/registry.ts#L331) | Mounts a plugin and creates a new awaitable Fiber on each call. |
| [`inject(deps, callback)`](../../../../vendor/cordis/src/registry.ts#L315) | Mounts the callback as a plugin and runs it when its required services are available. |
| [`resolve(plugin)`](../../../../vendor/cordis/src/registry.ts#L237) | Returns the entry point used as the registry key. A function or class resolves to itself, an object plugin resolves to its `apply` function, and a failed resolution returns `undefined`. |
| [`get(plugin)`](../../../../vendor/cordis/src/registry.ts#L251) | Looks up the shared `Runtime` and returns `undefined` if there is no record. |
| [`has(plugin)`](../../../../vendor/cordis/src/registry.ts#L262) | Checks whether the entry point has a `Runtime` record. |
| [`delete(plugin)`](../../../../vendor/cordis/src/registry.ts#L273) | Removes the `Runtime`, starts unloading all Fibers for the entry point, and returns the removed record immediately. Returns `undefined` if there is no record. |
| [`keys()`](../../../../vendor/cordis/src/registry.ts#L285), [`values()`](../../../../vendor/cordis/src/registry.ts#L290), [`entries()`](../../../../vendor/cordis/src/registry.ts#L295), [`forEach(callback)`](../../../../vendor/cordis/src/registry.ts#L304) | Iterate entry points, `Runtime` records, or `[callback, runtime]` pairs, or visit each record with arguments `(runtime, callback)`, respectively. |

Use `await fiber.dispose()` to unload a single mount and wait for cleanup. [Fiber cleanup](../../../../vendor/cordis/src/fiber.ts#L292)

When starting from a profile, the Loader imports the module by the YAML entry's `name` and calls `ctx.registry.plugin()`. The YAML `id` identifies the configuration entry and is available through `fiber.entry?.options.id`. [Loader mount](../../../../vendor/loader/src/config/entry.ts#L274) · [Configuration entry association](../../../../vendor/loader/src/index.ts#L151)

<a id="ctx-plugin"></a>
### `ctx.plugin()`

`ctx.plugin(plugin, config)` delegates to `RegistryService.plugin(plugin, config?, getOuterStack = buildOuterStack())`. `plugin` is the plugin definition, `config` is the raw configuration for this mount, and `getOuterStack` supplies the call stack for diagnostics. It defaults to the function returned by `buildOuterStack()`. [Method alias](../../../../vendor/cordis/src/reflect.ts#L233) · [Mount implementation](../../../../vendor/cordis/src/registry.ts#L331)

1. **Resolve the entry point.** A function or class serves directly as `callback`. An object plugin supplies its `apply` function. The call throws if the entry point is invalid or the current Fiber has been disposed. [Entry point resolution](../../../../vendor/cordis/src/registry.ts#L237) · [Mount checks](../../../../vendor/cordis/src/registry.ts#L331)
2. **Look up the shared record.** The registry uses the `callback` reference to look up a `Runtime`. The first registration creates a record with `name`, `callback`, `fibers`, and `Config`. [Runtime creation](../../../../vendor/cordis/src/registry.ts#L337)
3. **Create a Fiber for this mount.** The registry passes the current Context, raw configuration, resolved `inject` map, and shared `Runtime` to a new Fiber. The Fiber runs the plugin entry point once its dependencies are available. [Fiber creation](../../../../vendor/cordis/src/registry.ts#L345) · [Dependency-driven startup](../../../../vendor/cordis/src/fiber.ts#L639)

The method immediately returns an awaitable Fiber handle. `await ctx.plugin(...)` waits for the current lifecycle work to settle and throws if startup failed. If dependencies are still unavailable, the wait can finish with the Fiber in `PENDING`. [Return value](../../../../vendor/cordis/src/registry.ts#L345) · [Waiting for a Fiber](../../../../vendor/cordis/src/fiber.ts#L776)

<a id="ctx-inject"></a>
### `ctx.inject()`

Each call to `ctx.inject(deps, callback)` mounts a child plugin and creates a child Fiber for it. The child plugin declares its own service dependencies in `deps` and uses `callback` as its entry point. Through `RegistryService.inject()`, the call is equivalent to `ctx.plugin({ inject: deps, apply: callback, name: callback.name })`. Pass an array of service names, or use an object keyed by service name to include intercept configuration. [Implementation](../../../../vendor/cordis/src/registry.ts#L315) · [Dependency declaration type](../../../../vendor/cordis/src/registry.ts#L18)

In `tool-fs`, the module's exported `inject = ['tools', 'fs', 'systemPrompt', 'workingDirectory']` applies to the whole plugin. Cordis runs the module's `apply()` once all four services are available. When image reading is enabled, `apply()` declares a separate dependency for `read_image` with `ctx.inject(['attachments'], (imageCtx) => applyReadImageTool(imageCtx))`. The other file tools can register while `attachments` is unavailable. [Module dependencies](../../../../packages/fs/tool-fs/src/index.ts#L24) · [Image tool registration](../../../../packages/fs/tool-fs/src/index.ts#L70)

Cordis calls the callback with the child Fiber's Context once all dependencies are available. If any dependency becomes unavailable, Cordis cleans up the child Fiber's registrations and effects. When a dependency returns or is replaced, Cordis runs the callback again once all dependencies are available. Use the callback's Context to register tools and listeners and to attach `ctx.effect()` cleanup for resources that require explicit release. This ties cleanup to the feature's lifetime. [Dependency changes](../../../../vendor/cordis/src/fiber.ts#L652) · [Unload and reactivation](../../../../vendor/cordis/src/fiber.ts#L734)

`ctx.inject()` immediately returns an awaitable child Fiber. If a dependency is still unavailable, `await` can finish with the Fiber in `PENDING`. Call `await fiber.dispose()` to unload the feature explicitly. [Await semantics](#ctx-plugin) · [Fiber cleanup](../../../../vendor/cordis/src/fiber.ts#L292)

- `tool-bash` switches from a foreground-only tool to a version that supports background jobs when `jobs` becomes available. If `jobs` unloads while the parent plugin is still active, it restores the foreground-only version. [Source](../../../../packages/shell/tool-bash/src/index.ts#L588)
- `gateway` declares different dependencies in two calls. The RPC interceptor requires `connection`. WebSocket stream registration also requires `webServer`. [Source](../../../../packages/api/gateway/src/index.ts#L255)

<a id="fiber"></a>
## Fiber

Each `ctx.plugin(...)` mount creates a Fiber. During construction, `parent.extend({ fiber: this })` creates the plugin's `ctx`, which inherits from the mounting context and points `ctx.fiber` to the new Fiber. [Context creation](../../../../vendor/cordis/src/fiber.ts#L270)

### Core attributes

| Property | Purpose |
|---|---|
| `uid` | Registry identifier. The root Fiber uses `0`. An ordinary Fiber clears its identifier to `null` when disposal starts. |
| `parent` / `ctx` | The context used to mount the plugin and the context passed to the plugin, respectively. |
| `runtime` | The shared plugin record. The root Fiber has `null` here. |
| `inject` / `committed` / `provisions` | The required services, the implementations retained for this activation and its cleanup, and the implementations this Fiber provides, respectively. |
| `rawConfig` / `effectiveConfig` | Raw input and the resolved configuration used for the current or most recent activation or successful in-place update. Setting `rawConfig` only stores the input. |
| `state` / `inertia` | The current lifecycle state and the promise for an in-flight load or unload. |

### Core methods

| Method | Purpose |
|---|---|
| [`new Fiber(parent, config, inject, runtime, getOuterStack)`](../../../../vendor/cordis/src/fiber.ts#L256) | Initializes the root Fiber when `runtime` is `null`. For a plugin mount it creates a child context, registers cleanup with the parent Fiber, and checks initial dependencies. |
| [`effect(setup, label?)`](../../../../vendor/cordis/src/fiber.ts#L446) / [`getEffects()`](../../../../vendor/cordis/src/fiber.ts#L596) | Runs `setup` immediately and assigns the cleanup functions it produces to the current Fiber. The optional `label` is for diagnostics, and `getEffects()` returns metadata for registered effects. See [Using `effect()`](#fiber-effect) below. |
| [`await()`](../../../../vendor/cordis/src/fiber.ts#L776) | Waits for current lifecycle work and rethrows any startup error. It can return while the Fiber is `PENDING` for dependencies. |
| [`update(config)`](../../../../vendor/cordis/src/fiber.ts#L805) | Submits new configuration. The default is to restart, and an update hook can keep the current instance. |
| [`restart()`](../../../../vendor/cordis/src/fiber.ts#L790) | Requests a restart using the stored configuration. A running instance unloads before reloading. |
| [`dispose()`](../../../../vendor/cordis/src/fiber.ts#L292) | Permanently removes an ordinary Fiber and waits for cleanup. On the root Fiber it delegates to `restart()`. |
| [`resolveConfig(config)`](../../../../vendor/cordis/src/fiber.ts#L698) | Evaluates and validates a candidate configuration without applying it. |
| [`assertActive()`](../../../../vendor/cordis/src/fiber.ts#L379) | Throws once `uid` is `null`. Use `state === FiberState.ACTIVE` to check whether the plugin is running. |

These internal Fiber methods check dependencies and drive state transitions. The public `restart()` method increments `_revision` and requests a reload through `_refresh()`. `_unload()` cleans up the current instance, and `_reload()` starts the next one. [Restart entry point](../../../../vendor/cordis/src/fiber.ts#L790)

| Internal method | Purpose |
|---|---|
| [`_checkImpl(name)`](../../../../vendor/cordis/src/fiber.ts#L625) | Looks up the scoped service implementation from an `ACTIVE` Fiber and runs its optional `check`. Records the implementation in `_store` if available, or removes its entry otherwise. |
| [`_refresh()`](../../../../vendor/cordis/src/fiber.ts#L639) | Calculates the target `epoch` from `_revision` and the `uid` of each dependency implementation. Any missing required dependency makes the target `INACTIVE`. |
| [`_setEpoch(epoch)`](../../../../vendor/cordis/src/fiber.ts#L652) | Accepts a new lifecycle target. With no transition in progress, a change from `INACTIVE` to a valid target starts loading. Other target changes start unloading. |
| [`_reload()`](../../../../vendor/cordis/src/fiber.ts#L703) | Saves available dependencies in `committed` and records the dependency relationships. If the target `epoch` still matches, the call chain is `_reload() → _execute(this._runner) → _runner.execute()`. The runner resolves configuration, then calls `runtime.callback(ctx, config)` or creates a class plugin with `new runtime.callback(ctx, config)`. The entry point can register services and listeners through methods such as `ctx.provide()` and `ctx.on()`. A startup failure or target change leads to unloading. [Startup action](../../../../vendor/cordis/src/fiber.ts#L272) |
| [`_execute(runner)`](../../../../vendor/cordis/src/fiber.ts#L384) | Runs a plugin entry point or effect setup function and collects the cleanup functions it returns or yields. |
| [`_unload()`](../../../../vendor/cordis/src/fiber.ts#L734) | Waits for Fibers that use its services to unload, runs its cleanup functions, and releases its own service dependencies. After cleanup, the current `epoch` determines whether to reload. |
| [`_getState()`](../../../../vendor/cordis/src/fiber.ts#L602) | Derives a stable state from `uid`, the startup error, and `epoch`, checking for `DISPOSED`, `FAILED`, `ACTIVE`, and `PENDING` in that order. |
| [`_updateState(callback)`](../../../../vendor/cordis/src/fiber.ts#L609) | Sets the state supplied by the callback or derives it with `_getState()`. Emits `internal/status` when the state changes and notifies service dependents when entering or leaving `ACTIVE`. |

[Effect registration](../../../../vendor/cordis/src/fiber.ts#L443) · [Configuration access](../../../../vendor/cordis/src/fiber.ts#L670) · [Lifecycle methods](../../../../vendor/cordis/src/fiber.ts#L771)

<a id="fiber-effect"></a>
#### Using `effect()`

Plugins commonly call `ctx.effect()` in their entry point, such as `apply(ctx)`, so resources are released when the plugin unloads. They can also call it while running. For example, `LoggerService.exporter()` calls `ctx.effect()` for each exporter registration, assigning the registration to the caller's Fiber. The plugin entry point or service method decides when to register the effect. The `setup` passed to `ctx.effect()` performs the operation and supplies cleanup. [Log exporter registration](../../../../vendor/cordis/src/logger.ts#L235)

- **Synchronous setup.** Use `const dispose = ctx.effect(setup)`, where `setup` returns a cleanup function directly. Call `await dispose()` to clean up early. Synchronous setup can also supply an asynchronous cleanup function. [Synchronous overload](../../../../vendor/cordis/src/fiber.ts#L443) · [Asynchronous cleanup example](../../../../packages/session/session-persistence-sqlite/src/sqlite.ts#L985)
- **Asynchronous setup.** Use `const dispose = await ctx.effect(asyncSetup)` to wait for setup and obtain the cleanup function. For example, the directory picker returns `unmount` after creating and awaiting two plugins. A later call to `await dispose()` runs cleanup. [Waiting for setup](../../../../vendor/cordis/src/fiber.ts#L578) · [Directory picker example](../../../../packages/host/directory-picker-auto/src/index.ts#L78)
- **Multiple cleanup actions.** Use `ctx.effect(function* () { … })` and `yield` each cleanup function. Cleanup functions within one effect run in reverse order. For example, `command-compact` unregisters the command before waiting for invocations already in progress. [Command registration example](../../../../packages/compaction/command-compact/src/index.ts#L98)

<a id="fiber-lifecycle"></a>
### Lifecycle and cleanup

When dependencies are ready, a Fiber moves from `PENDING` through `LOADING` to `ACTIVE`. When a dependency disappears, the Fiber returns to `PENDING` through `UNLOADING`. After a startup failure, Cordis cleans up the activation before entering `FAILED`. Explicit disposal of an ordinary Fiber ends in `DISPOSED`. [State transitions](../../../../vendor/cordis/src/fiber.ts#L639)

A child Fiber registers its cleanup through `parent.fiber.effect(...)`. For example, `workspace-controller` mounts `DirectoryPickerController` through its own `ctx`, so unloading the parent also cleans up the child. Service dependencies also determine teardown order. A service provider waits for Fibers using its service to unload before cleaning up its own effects. [Child Fiber cleanup registration](../../../../vendor/cordis/src/fiber.ts#L292) · [`DirectoryPickerController`](../../../../packages/api/workspace-controller/src/index.ts#L82) · [Dependency teardown order](../../../../vendor/cordis/src/fiber.ts#L734)

<a id="fiber-stable-transitions"></a>
#### Stable state transitions

An ordinary plugin Fiber starts in `PENDING`. The stable states are `PENDING`, `ACTIVE`, `FAILED`, and `DISPOSED`. Each row starts and ends in a stable state, and its full path includes the intervening `LOADING` and `UNLOADING` states.

| Start | End | Trigger | Full path | Functions and source |
|---|---|---|---|---|
| `PENDING` | `ACTIVE` | All dependencies become available and startup succeeds. | `PENDING → LOADING → ACTIVE` | [`_refresh()` calculates the target](../../../../vendor/cordis/src/fiber.ts#L639) · [`_setEpoch()` starts loading](../../../../vendor/cordis/src/fiber.ts#L660) · [`_reload()` finishes loading](../../../../vendor/cordis/src/fiber.ts#L724) |
| `PENDING` | `FAILED` | Initial startup fails and cleanup finishes. | `PENDING → LOADING → UNLOADING → FAILED` | [`_reload()` records the error and starts unloading](../../../../vendor/cordis/src/fiber.ts#L715) · [`_unload()` finishes cleanup](../../../../vendor/cordis/src/fiber.ts#L760) · [`_getState()` selects `FAILED`](../../../../vendor/cordis/src/fiber.ts#L604) |
| `PENDING` | `PENDING` | A dependency becomes unavailable during startup and remains unavailable after cleanup. | `PENDING → LOADING → UNLOADING → PENDING` | [`_setEpoch()` updates the target](../../../../vendor/cordis/src/fiber.ts#L657) · [`_reload()` starts unloading](../../../../vendor/cordis/src/fiber.ts#L724) · [`_unload()` finishes cleanup](../../../../vendor/cordis/src/fiber.ts#L760) |
| `ACTIVE` | `PENDING` | A dependency becomes unavailable and remains unavailable after cleanup. | `ACTIVE → UNLOADING → PENDING` | [`_refresh()` calculates `INACTIVE`](../../../../vendor/cordis/src/fiber.ts#L639) · [`_setEpoch()` starts unloading](../../../../vendor/cordis/src/fiber.ts#L663) · [`_unload()` finishes cleanup](../../../../vendor/cordis/src/fiber.ts#L760) |
| `ACTIVE` | `ACTIVE` | A restart or a change in dependency implementation leads to a successful reload. | `ACTIVE → UNLOADING → LOADING → ACTIVE` | [`restart()` increments `_revision`](../../../../vendor/cordis/src/fiber.ts#L790) · [`_refresh()` calculates the target](../../../../vendor/cordis/src/fiber.ts#L639) · [`_setEpoch()` starts unloading](../../../../vendor/cordis/src/fiber.ts#L663) · [`_unload()` starts the next load](../../../../vendor/cordis/src/fiber.ts#L760) · [`_reload()` finishes loading](../../../../vendor/cordis/src/fiber.ts#L724) |
| `ACTIVE` | `FAILED` | The reload fails and cleanup finishes. | `ACTIVE → UNLOADING → LOADING → UNLOADING → FAILED` | [`_unload()` starts the next load](../../../../vendor/cordis/src/fiber.ts#L760) · [`_reload()` records the error and starts unloading](../../../../vendor/cordis/src/fiber.ts#L715) · [`_getState()` selects `FAILED`](../../../../vendor/cordis/src/fiber.ts#L604) |
| `FAILED` | `ACTIVE` | `update(config)` clears the error and the retry succeeds. | `FAILED → LOADING → ACTIVE` | [`update()` clears the error](../../../../vendor/cordis/src/fiber.ts#L812) · [`update()` starts a retry](../../../../vendor/cordis/src/fiber.ts#L827) · [`_reload()` finishes loading](../../../../vendor/cordis/src/fiber.ts#L724) |
| `FAILED` | `FAILED` | `update(config)` starts a retry, and startup fails again. | `FAILED → LOADING → UNLOADING → FAILED` | [`update()` starts a retry](../../../../vendor/cordis/src/fiber.ts#L827) · [`_reload()` records the new error](../../../../vendor/cordis/src/fiber.ts#L715) · [`_getState()` selects `FAILED`](../../../../vendor/cordis/src/fiber.ts#L604) |
| `FAILED` | `PENDING` | A dependency becomes unavailable during a retry and remains unavailable after cleanup. | `FAILED → LOADING → UNLOADING → PENDING` | [`update()` starts a retry](../../../../vendor/cordis/src/fiber.ts#L827) · [`_reload()` starts unloading](../../../../vendor/cordis/src/fiber.ts#L724) · [`_unload()` finishes cleanup](../../../../vendor/cordis/src/fiber.ts#L760) |
| `PENDING` | `DISPOSED` | The Fiber is disposed while it is waiting for dependencies. | `PENDING → UNLOADING → DISPOSED` | [`dispose()` starts cleanup directly](../../../../vendor/cordis/src/fiber.ts#L308) · [`_getState()` checks `uid`](../../../../vendor/cordis/src/fiber.ts#L603) |
| `ACTIVE` | `DISPOSED` | A running plugin is disposed. | `ACTIVE → UNLOADING → DISPOSED` | [`dispose()` clears `uid`](../../../../vendor/cordis/src/fiber.ts#L294) · [`_setEpoch()` starts unloading](../../../../vendor/cordis/src/fiber.ts#L663) · [`_getState()` checks `uid`](../../../../vendor/cordis/src/fiber.ts#L603) |
| `FAILED` | `DISPOSED` | The Fiber is disposed after a startup failure. | `FAILED → UNLOADING → DISPOSED` | [`dispose()` starts cleanup directly](../../../../vendor/cordis/src/fiber.ts#L308) · [`_getState()` checks `uid`](../../../../vendor/cordis/src/fiber.ts#L603) |

If the target `epoch` changes again during loading or unloading, the Fiber can repeat `LOADING → UNLOADING` before it settles. `DISPOSED` ends an ordinary Fiber's lifecycle. The root Fiber starts in `ACTIVE`, and its `dispose()` delegates to `restart()`. [End of loading](../../../../vendor/cordis/src/fiber.ts#L724) · [End of cleanup](../../../../vendor/cordis/src/fiber.ts#L760) · [Root Fiber](../../../../vendor/cordis/src/fiber.ts#L348)

<a id="fiber-construction"></a>
#### Construction

An ordinary Fiber creates the plugin context and registers cleanup before it first checks dependencies. Listeners that receive `internal/plugin` can add dependencies or begin disposing the Fiber. [Constructor](../../../../vendor/cordis/src/fiber.ts#L256)

1. **Prepare arguments.** For `ctx.plugin(plugin, config)`, the registry resolves the plugin entry point, obtains the shared `runtime`, and normalizes `inject` into a dependency map. It passes the mounting Context to `new Fiber(...)` as `parent`. A child plugin mounted through `ctx.inject()` follows the same path. [`RegistryService.inject()`](../../../../vendor/cordis/src/registry.ts#L315) · [Fiber creation](../../../../vendor/cordis/src/registry.ts#L331)
2. **Create the plugin context.** The constructor stores the raw configuration, assigns a `uid`, and creates the plugin Context with `parent.extend({ fiber: this })`. At this point, `ctx.fiber` points to the new Fiber, `parent.fiber` points to its parent Fiber, and the new Fiber is `PENDING`. [Context creation](../../../../vendor/cordis/src/fiber.ts#L265)
3. **Prepare startup and register cleanup.** `_runner.execute` holds the action that resolves configuration and calls the plugin entry point. Through `parent.fiber.effect(...)`, the constructor adds the new Fiber to `runtime.fibers` and sets its `dispose()`. [Startup action](../../../../vendor/cordis/src/fiber.ts#L272) · [Cleanup registration](../../../../vendor/cordis/src/fiber.ts#L292)
4. **Emit `internal/plugin`.** The Loader's synchronous listener associates `fiber.entry` and merges the YAML entry's `inject` into `fiber.inject`. Other listeners can call `fiber.dispose()` at this point. The constructor skips the first dependency check if the `uid` has been cleared or the parent Fiber is unloading. [Event emission](../../../../vendor/cordis/src/fiber.ts#L327) · [Loader dependency merge](../../../../vendor/loader/src/index.ts#L149)
5. **Check dependencies.** `_checkImpl(name)` looks up an available implementation for each dependency. `_refresh()` uses the dependencies and restart revision to calculate the target `epoch`. The Fiber waits while a dependency is missing. Once all are available, it enters `LOADING`, runs the startup action, and enters `ACTIVE` on success. [First dependency check](../../../../vendor/cordis/src/fiber.ts#L338) · [State transitions](../../../../vendor/cordis/src/fiber.ts#L639)

For a class plugin, the startup action runs the class constructor, the callbacks in `initHooks`, and the optional `[Service.init]()`, in that order. A method-level `@Inject` hook uses `ctx.inject()` to create a child Fiber, which calls the original method once its own dependencies are available. [Class plugin startup](../../../../vendor/cordis/src/fiber.ts#L278) · [Method decorator](../../../../vendor/cordis/src/registry.ts#L54)

`new Context()` passes `runtime = null` when it creates the root Fiber. The constructor uses the root Context directly, sets `uid` to `0`, and starts in `ACTIVE`. [Root Fiber branch](../../../../vendor/cordis/src/fiber.ts#L348)

<a id="fiber-dependency-resolution"></a>
#### Dependency resolution

Suppose B's Fiber is waiting for the `fs` service that A provides. When the service becomes available, `ReflectService.notify()` makes B reevaluate whether it can start.

1. **The provider announces service availability.** If A calls `ctx.provide('fs', ...)` during startup, Cordis registers the implementation and announces it when A enters `ACTIVE`. If A is already `ACTIVE`, `provide()` announces it immediately. [Provider state changes](../../../../vendor/cordis/src/fiber.ts#L609) · [Service registration](../../../../vendor/cordis/src/reflect.ts#L324)
2. **`notify()` finds affected Fibers.** It scans the registry for Fibers that declare `fs` in `inject` and share the same isolation label for the service. It directly calls `_checkImpl('fs')` and `_refresh()` on those Fibers. [Service notification](../../../../vendor/cordis/src/reflect.ts#L357)
3. **B checks all dependencies.** `_checkImpl()` confirms that the provider is `ACTIVE` and runs the implementation's optional `check`. Once the implementation passes, the method records it in B's `_store`. `_refresh()` checks all dependencies against that map. If `fs` was the last missing dependency, the target `epoch` changes from `INACTIVE` to a valid value, and `_setEpoch()` calls `_reload()` to put B in `LOADING`. [Dependency checks and startup](../../../../vendor/cordis/src/fiber.ts#L625)

`notify()` triggers startup by calling the Fiber's dependency checks directly. After that pass, it emits `internal/service` to let listeners know the service changed. [Event emission](../../../../vendor/cordis/src/reflect.ts#L371)

<a id="fiber-dependency-loss"></a>
#### Dependency loss

Suppose B uses the `fs` service that A provides. If `fs` becomes unavailable, B unloads its current instance. During cleanup, B can still use the implementation bound at startup through `ctx.fs`. [Service lookup](../../../../vendor/cordis/src/reflect.ts#L147)

1. **The service becomes unavailable.** When A leaves `ACTIVE`, it notifies dependents to check the service again. Dependency checks require an `ACTIVE` provider even if the implementation is still registered. Calling the disposer returned by `ctx.provide('fs', ...)` removes the implementation from the service store before notifying dependents. [State change notification](../../../../vendor/cordis/src/fiber.ts#L609) · [Availability check](../../../../vendor/cordis/src/reflect.ts#L273) · [Service removal](../../../../vendor/cordis/src/reflect.ts#L333)
2. **B enters `UNLOADING`.** `notify()` finds B because B requires `fs` and shares its isolation label, then calls `_checkImpl('fs')`. If no implementation is available, B removes the entry from `_store`. `_refresh()` sets the target `epoch` to `INACTIVE`, and `_setEpoch()` starts unloading. Any Fibers using services that B provides are also notified when B leaves `ACTIVE`. [Affected Fibers](../../../../vendor/cordis/src/reflect.ts#L357) · [Target state refresh](../../../../vendor/cordis/src/fiber.ts#L625) · [Downstream notification](../../../../vendor/cordis/src/fiber.ts#L609)
3. **Keep the previous implementation through cleanup.** `_store` tracks implementations available for the next startup, and `committed` holds those bound at the current startup. Cleanup functions can still read B's `committed.fs` through `ctx.fs`. If A is also unloading, it waits for B to release the dependency before running its own cleanup functions. After cleanup, B removes itself from the old implementation's `dependents` set and clears `committed`. [Bound service lookup](../../../../vendor/cordis/src/reflect.ts#L161) · [Unload and dependency release](../../../../vendor/cordis/src/fiber.ts#L734)
4. **Choose the next state using the latest dependencies.** If a dependency is still missing, B enters `PENDING` and keeps the same Fiber to receive service notifications. If all dependencies become available during cleanup, B enters `LOADING` as soon as cleanup finishes and starts again with the implementations now available. [State after unloading](../../../../vendor/cordis/src/fiber.ts#L760)

<a id="reflect-service"></a>
## ReflectService

`ReflectService` implements scoped service lookup and exposes service methods on the Context. [Source](../../../../vendor/cordis/src/reflect.ts#L244)

### Core attributes

| Property | Purpose |
|---|---|
| `store` | Registered service implementations indexed by their scope labels. |
| `props` | Definitions for properties exposed on the Context. |

### Core methods

| Method | Purpose |
|---|---|
| [`new ReflectService(ctx)`](../../../../vendor/cordis/src/reflect.ts#L249) | Binds the reflection service to a Context and installs the built-in aliases for reflection, Fiber, registry, and events. |
| [`get(name)`](../../../../vendor/cordis/src/reflect.ts#L269) | Looks up a service in the caller's scope. |
| [`provide(name, value?, check?)`](../../../../vendor/cordis/src/reflect.ts#L313) | Registers a service implementation and ties its cleanup to the caller's Fiber. |
| [`accessor(name, options)`](../../../../vendor/cordis/src/reflect.ts#L385) / [`mixin(source, mixins)`](../../../../vendor/cordis/src/reflect.ts#L404) | Expose an accessor or selected service members on the Context. |

<a id="events-service"></a>
## EventsService

`EventsService` registers and dispatches listeners. [Source](../../../../vendor/cordis/src/events.ts#L130)

### Core attributes

| Property | Purpose |
|---|---|
| `_hooks` | Internal table of listeners by event name. Each registration records its Context. |

### Core methods

| Method | Purpose |
|---|---|
| [`new EventsService(ctx)`](../../../../vendor/cordis/src/events.ts#L133) | Binds the event service to a Context and registers the built-in listener, update, and configuration handlers. |
| [`on(name, listener)`](../../../../vendor/cordis/src/events.ts#L298) / [`once(name, listener)`](../../../../vendor/cordis/src/events.ts#L321) | Register a recurring or one-shot listener under the current Fiber. |
| [`emit(...)`](../../../../vendor/cordis/src/events.ts#L207) / [`parallel(...)`](../../../../vendor/cordis/src/events.ts#L193) | `emit` invokes listeners synchronously. `parallel` runs listeners concurrently and waits for all to settle. |
| [`bail(...)`](../../../../vendor/cordis/src/events.ts#L232) / [`serial(...)`](../../../../vendor/cordis/src/events.ts#L218) | Stop at the first return value other than `undefined`, `null`, or `false`. `serial` awaits each listener. |
| [`waterfall(...)`](../../../../vendor/cordis/src/events.ts#L250) | Wraps a final callback in listeners. Each listener calls `next()` to continue the chain. |

<a id="logger-service"></a>
## LoggerService

`LoggerService` delivers structured log messages to registered exporters. [Source](../../../../vendor/cordis/src/logger.ts#L191)

### Core attributes

| Property | Purpose |
|---|---|
| `buffer` / `bufferSize` | Recent log messages and the maximum number retained in the buffer. |
| `exporters` | Registered log exporters. |

### Core methods

| Method | Purpose |
|---|---|
| [`new LoggerService(ctx)`](../../../../vendor/cordis/src/logger.ts#L202) | Creates the callable logging service and installs the default exporter that retains recent messages in the buffer. |
| [`ctx.logger(name?)`](../../../../vendor/cordis/src/logger.ts#L243) | Creates a logger. Without an explicit name, it uses the configured name or derives one from the caller's Fiber. |
| [`error()`](../../../../vendor/cordis/src/logger.ts#L256) / [`warn()`](../../../../vendor/cordis/src/logger.ts#L256) / [`info()`](../../../../vendor/cordis/src/logger.ts#L256) / [`debug()`](../../../../vendor/cordis/src/logger.ts#L256) | Write log messages at the corresponding levels. |
| [`exporter(exporter)`](../../../../vendor/cordis/src/logger.ts#L235) | Registers a log exporter and returns a disposer. The current Fiber also cleans up the registration when it unloads. |
