# Cordis Context 与核心服务

[English](cordis_context.md) | 中文

## 摘要

沿着 `ctx.plugin()`、`ctx.provide()` 或 `ctx.on()` 等调用，从插件的 Context 找到具体实现。理解 Cordis 支持的插件入口，以及 Registry 如何为每次挂载创建 Fiber。

## 目录

- [核心对象](#core-objects)
  - [Context](#context)
- [Plugin](#plugin)
  - [`Plugin.Runtime`](#plugin-runtime)
  - [`Plugin.inject`](#plugin-inject)
- [Registry](#registry)
  - [`ctx.plugin()`](#ctx-plugin)
  - [`ctx.inject()`](#ctx-inject)
- [Fiber](#fiber)
  - [生命周期与清理](#fiber-lifecycle)
    - [稳定状态转换](#fiber-stable-transitions)
    - [构造](#fiber-construction)
    - [依赖解决](#fiber-dependency-resolution)
    - [失去依赖](#fiber-dependency-loss)
- [ReflectService](#reflect-service)
- [EventsService](#events-service)
- [LoggerService](#logger-service)

<a id="core-objects"></a>
## 核心对象

`new Context()` 创建根 Fiber 和内置服务，然后返回一个 Proxy。`ReflectService` 在 `ctx` 上设置方法的转发入口，因此 `ctx.plugin()` 交给注册器，`ctx.provide()` 交给服务注册表，`ctx.on()` 交给事件服务。[Context 构造函数](../../../../vendor/cordis/src/context.ts#L73) · [方法转发](../../../../vendor/cordis/src/reflect.ts#L233)

| 对象 | 职责 |
|---|---|
| `Context` | 给插件提供统一的调用入口和服务查找作用域。子上下文继承父上下文，`ctx.isolate(name)` 可以创建一个独立查找指定服务名的子作用域。[Context](../../../../vendor/cordis/src/context.ts#L42) · [`isolate()`](../../../../vendor/cordis/src/context.ts#L126) |
| `ReflectService` · `ctx.reflect` | 通过 `provide()` 注册服务实现。读取 `ctx.webServer` 之类的属性时，它的 Proxy 拦截器会按调用者的上下文和服务作用域查找。[服务注册](../../../../vendor/cordis/src/reflect.ts#L313) · [属性查找](../../../../vendor/cordis/src/reflect.ts#L144) |
| `RegistryService` · `ctx.registry` | 找到插件入口，保存插件运行时记录，并为每次 `ctx.plugin()` 注册创建一个 Fiber。[插件注册](../../../../vendor/cordis/src/registry.ts#L331) |
| `Fiber` · `ctx.fiber` | 管理一个插件实例。它跟踪依赖和配置，激活插件，并在插件卸载时执行通过 `ctx.effect()` 注册的清理函数。[Fiber](../../../../vendor/cordis/src/fiber.ts#L214) · [`effect()`](../../../../vendor/cordis/src/fiber.ts#L443) |
| `EventsService` · `ctx.events` | 通过 `ctx.on()` 注册监听器，并用 `emit`、`parallel`、`waterfall` 等模式分发事件。监听器的注册归所属插件的 Fiber 管理。[EventsService](../../../../vendor/cordis/src/events.ts#L130) · [监听器注册](../../../../vendor/cordis/src/events.ts#L298) |
| `LoggerService` · `ctx.logger` | 把日志发给已注册的接收器。Host 启动时，`boot()` 注册一个接收器，收集警告和错误供诊断使用。[LoggerService](../../../../vendor/cordis/src/logger.ts#L194) · [启动日志接收器](../../../../packages/boot/app-boot/src/index.ts#L1280) |

<a id="context"></a>
### Context

`Context` 为插件代码提供当前作用域和内置服务。 [源码](../../../../vendor/cordis/src/context.ts#L42)

#### 核心属性

| 属性 | 用途 |
|---|---|
| `fiber` | 管理通过当前上下文注册的副作用。根上下文指向根 Fiber，插件上下文指向本次挂载创建的 Fiber。 |
| `runtime` | `ctx.fiber.runtime` 的转发属性，指向插件入口的共享运行时记录。根 Fiber 的值为 `null`。 |
| `baseUrl` | 可选的基础 URL。Loader 可以用它解析插件模块的相对路径。 |
| `reflect` / `registry` | `reflect` 查找和注册服务。`registry` 查找插件入口，并为每次挂载创建 Fiber。 |
| `events` / `logger` | `events` 注册和分发监听器。`logger` 可通过 `ctx.logger.info(...)` 直接记录日志，也可通过 `ctx.logger(name)` 创建具名日志记录器。 |
| `ctx[Context.root]` | 整棵树共用的根 Context。要找发起当前插件挂载的上下文，读取 `ctx.fiber.parent`。 |
| `ctx[Context.isolate]` / `ctx[Context.intercept]` | 分别把服务名映射到作用域标签和拦截元数据。`isolate()` 和 `intercept()` 创建带有相应条目的子上下文。 |
| `ctx[Context.filter]` / `disposer[Context.effect]` | 当前上下文作为事件分发的接收者传入时，可通过可选过滤函数筛选监听器。`Context.effect` 是清理函数上用于读取副作用诊断信息的 symbol 键。 |

#### 核心方法

创建子上下文和检查 Context 类型的方法定义在 `Context` 类上。 [类方法源码](../../../../vendor/cordis/src/context.ts#L54)

| 方法 | 用途 |
|---|---|
| [`new Context()`](../../../../vendor/cordis/src/context.ts#L73) | 创建根 Fiber 和内置服务，再返回根 Context 的 Proxy。 |
| [`extend(meta?)`](../../../../vendor/cordis/src/context.ts#L101) | 创建继承当前上下文的子上下文。`meta` 的自有属性（包括 symbol 键）在子上下文上优先。子上下文默认继承当前 Fiber，也可以通过 `meta` 显式指定 Fiber。 |
| [`isolate(name, label?)`](../../../../vendor/cordis/src/context.ts#L126) | 创建子上下文，使指定服务的查找与注册使用 `label`。省略标签时会创建新的 symbol。复用同一个标签可以共享作用域。隔离作用域中的插件若直接读取尚不可用的必需服务，读取会抛错。 |
| [`intercept(name, config)`](../../../../vendor/cordis/src/context.ts#L147) | 创建携带指定服务配置的子上下文。服务解析拦截配置时，会合并依赖声明和各层上下文提供的元数据。 |
| [`Context.is(value)`](../../../../vendor/cordis/src/context.ts#L63) | 判断值是否为 Cordis Context，并收窄 TypeScript 类型。检查使用全局 symbol 标记，可跨 Cordis 副本识别。 |

`ReflectService` 把下面的方法转发给当前上下文里的 Fiber 或内置服务。类型声明 `interface Context extends Pick<Fiber, 'effect'>` 让 `ctx.effect` 沿用 `Fiber.effect` 的签名，运行时转发由 `mixin('fiber', ['runtime', 'effect'])` 安装。 [类型声明](../../../../vendor/cordis/src/fiber.ts#L9) · [转发注册](../../../../vendor/cordis/src/reflect.ts#L255)

| 方法 | 用途 |
|---|---|
| [`effect(setup, label?)`](../../../../vendor/cordis/src/fiber.ts#L446) | 调用 `ctx.fiber.effect()`，立即运行 `setup`。返回的清理函数可以在所属 Fiber 卸载前主动释放这个副作用。 |
| [`plugin(plugin, config?)`](../../../../vendor/cordis/src/registry.ts#L331) | 在当前 Fiber 下挂载插件，返回可等待的子 Fiber 句柄。 |
| [`inject(deps, callback)`](../../../../vendor/cordis/src/registry.ts#L315) | 挂载一个子 Fiber。所需服务齐备后，用子 Fiber 的上下文调用回调。 |
| [`get(name, strict?)`](../../../../vendor/cordis/src/reflect.ts#L269) / [`set(name, value)`](../../../../vendor/cordis/src/reflect.ts#L290) | `get()` 在当前作用域查找服务，无需先声明 `inject`。找不到匹配的实现时返回 `undefined`，默认只返回已激活的提供方的实现。`set()` 替换当前 Fiber 已注册的服务值。没有注册或注册属于其他 Fiber 时，调用会抛错。 |
| [`provide(name, value?, check?)`](../../../../vendor/cordis/src/reflect.ts#L313) | 按当前作用域标签注册服务，并返回清理函数。可选的 `check` 判断这个实现是否可供依赖方使用。 |
| [`accessor(name, options)`](../../../../vendor/cordis/src/reflect.ts#L385) / [`mixin(source, mixins)`](../../../../vendor/cordis/src/reflect.ts#L404) | 在 `ctx` 上定义计算属性，或直接暴露另一个上下文属性的指定成员。两者都返回清理函数。 |
| [`on(name, listener)`](../../../../vendor/cordis/src/events.ts#L298) / [`once(name, listener)`](../../../../vendor/cordis/src/events.ts#L321) | 注册可重复触发或只触发一次的监听器，并返回取消订阅函数。注册归当前 Fiber 管理。 |
| [`emit()`](../../../../vendor/cordis/src/events.ts#L207) / [`parallel()`](../../../../vendor/cordis/src/events.ts#L193) / [`bail()`](../../../../vendor/cordis/src/events.ts#L232) / [`serial()`](../../../../vendor/cordis/src/events.ts#L218) / [`waterfall()`](../../../../vendor/cordis/src/events.ts#L250) | 按 `EventsService` 的相应模式分发事件。 |

例如，`const child = ctx.isolate('cache')` 给 `cache` 分配了独立作用域，但继承了 `ctx.fiber`。调用 `child.plugin(...)` 才会创建新的 Fiber，新插件在 `child` 的作用域中查找 `cache`。 [作用域创建](../../../../vendor/cordis/src/context.ts#L126) · [Fiber 创建](../../../../vendor/cordis/src/registry.ts#L345)

<a id="plugin"></a>
## Plugin

`Plugin<T>` 定义了 `ctx.plugin()` 接受的三种入口。三种入口都接收插件自己的 Context 和配置，`T` 表示入口收到的配置类型。[Plugin 类型定义](../../../../vendor/cordis/src/registry.ts#L107)

| 类型 | 启动方式 |
|---|---|
| `Plugin.Function<T>` | 直接调用函数 `plugin(ctx, config)`。 |
| `Plugin.Constructor<T>` | 用 `new PluginClass(ctx, config)` 创建实例。 |
| `Plugin.Object<T>` | 执行对象的 `apply(ctx, config)` 入口。 |

三种入口共用 `Plugin.Base<T>` 定义的可选字段。[插件元数据](../../../../vendor/cordis/src/registry.ts#L115)

| 字段 | 类型 | 用途 |
|---|---|---|
| `name` | `string` | 用于诊断。函数和类通常使用 JavaScript 提供的 `.name`，对象插件可以显式填写。 |
| `Config` | `StandardSchemaV1<any, T>` | 定义如何校验和转换配置。插件入口收到 schema 返回的配置。 |
| `inject` | `Inject` | 声明启动所需的服务名。Fiber 在当前作用域按这些名称查找可用的服务。对象写法还可以为各项服务附带拦截配置。[依赖检查](../../../../vendor/cordis/src/fiber.ts#L625) |
| `provide` | `string` 或 `string[]` | **当前仓库的插件均未使用此字段。** 普通插件的入口可调用 `ctx.provide(name, value)` 注册服务。[插件挂载](../../../../vendor/loader/src/config/entry.ts#L274) · [服务注册](../../../../vendor/cordis/src/reflect.ts#L313) |
| `intercept` | `Dict<boolean>` | 声明插件会读取哪些服务的拦截配置。**当前仓库的插件均未声明此字段。** [字段定义](../../../../vendor/cordis/src/registry.ts#L126) |

大写 `Config` 是插件定义上的 schema，Registry 把它保存在共享的 `Runtime` 里。小写 `config` 是 `ctx.plugin(plugin, config)` 传入的本次挂载配置，由该次挂载的 Fiber 保存。Fiber 启动时解析配置，再调用插件入口。[Registry 注册](../../../../vendor/cordis/src/registry.ts#L331) · [Fiber 启动](../../../../vendor/cordis/src/fiber.ts#L273)

Loader 可以把导出 `apply` 和 `inject` 的模块作为对象插件挂载。模块级依赖和插件内的局部依赖可以同时使用。参见 [`ctx.inject()` 的 `tool-fs` 示例](#ctx-inject)。

<a id="plugin-runtime"></a>
### `Plugin.Runtime`

`Plugin.Runtime` 是同一个插件入口的各次挂载共享的记录。Registry 用 `Map<Function, Plugin.Runtime>` 保存这些记录，键是入口函数的引用。两个对象插件如果使用同一个 `apply` 函数，也会共享一条记录。[注册表定义](../../../../vendor/cordis/src/registry.ts#L212) · [入口解析](../../../../vendor/cordis/src/registry.ts#L237)

| 字段 | 用途 |
|---|---|
| `name` | 首次注册时取自插件定义，用于诊断。 |
| `callback` | 共享的可执行入口，也是 Registry 查找记录的键。 |
| `fibers` | 保存该入口当前挂载的 Fiber，包括仍在等待依赖的挂载。 |
| `Config` | 首次注册时取自插件定义的可选 schema，供各个 Fiber 解析自己的配置。 |

每次调用 `ctx.plugin()` 都会创建新的 Fiber，各自保存原始配置、Context 和生命周期状态。单独卸载一个 Fiber 时，Registry 会把它从 `fibers` 中移除。移除最后一个 Fiber 时，Registry 会删除这条 `Runtime` 记录。[Runtime 定义](../../../../vendor/cordis/src/registry.ts#L151) · [创建 Fiber](../../../../vendor/cordis/src/registry.ts#L345) · [移除 Fiber](../../../../vendor/cordis/src/fiber.ts#L292)

<a id="plugin-inject"></a>
### `Plugin.inject`

| 声明位置 | 写法 | 谁等待依赖 |
|---|---|---|
| 模块插件 | 与 `apply` 一起导出 `export const inject = ['fs']`。 | 插件 Fiber。[示例](../../../../packages/fs/tool-fs/src/index.ts#L24) |
| 对象、类或函数插件 | 对象写 `{ inject: ['fs'], apply }`，类写 `static inject = ['fs']`，函数可以设置 `apply.inject = ['fs']`。 | 插件 Fiber。[挂载实现](../../../../vendor/cordis/src/registry.ts#L331) |
| 类装饰器 | 在类上写 `@Inject('loader')`，可以叠加。 | 插件 Fiber。装饰器把依赖加入类的 `static inject`。[示例](../../../../packages/boot/hmr/src/index.ts#L211) |
| 方法装饰器 | 在类方法上写 `@Inject('timer')`，可以叠加。 | 该方法对应的子 Fiber。方法声明的依赖齐备后，Cordis 才调用它。[实现](../../../../vendor/cordis/src/registry.ts#L54) |
| Loader 的 YAML 条目 | 在条目上写 `inject: [fs]`。 | 该次挂载的 Fiber。Loader 在 `internal/plugin` 回调里把声明合并到 `fiber.inject`，随后 Fiber 才检查依赖。同名项以 YAML 的声明为准。[合并位置](../../../../vendor/loader/src/index.ts#L154) · [合并规则](../../../../vendor/cordis/src/registry.ts#L86) |

<a id="registry"></a>
## Registry

Registry 管理插件的挂载记录。以下方法通过 `ctx.registry` 调用，Context 也直接提供 `ctx.plugin()` 和 `ctx.inject()`。[RegistryService 源码](../../../../vendor/cordis/src/registry.ts#L210)

### 核心属性

| 属性 | 用途 |
|---|---|
| `size` | 当前 `Runtime` 记录数。 |
| `counter` | 每次读取都会递增，并分配下一个 Fiber 编号。 |

### 核心方法

| 方法 | 用途 |
| --- | --- |
| [`new RegistryService(ctx)`](../../../../vendor/cordis/src/registry.ts#L214) | 将注册器关联到 Context。运行时记录表初始为空，编号计数器从零开始。 |
| [`plugin(plugin, config?)`](../../../../vendor/cordis/src/registry.ts#L331) | 挂载插件，每次调用都创建并返回一个可等待的 Fiber。 |
| [`inject(deps, callback)`](../../../../vendor/cordis/src/registry.ts#L315) | 将回调挂载为插件，等所需服务就绪后运行。 |
| [`resolve(plugin)`](../../../../vendor/cordis/src/registry.ts#L237) | 取得用作注册键的入口函数。函数和类返回自身，对象插件返回其 `apply`，无法解析时返回 `undefined`。 |
| [`get(plugin)`](../../../../vendor/cordis/src/registry.ts#L251) | 查询共享的 `Runtime`，没有记录时返回 `undefined`。 |
| [`has(plugin)`](../../../../vendor/cordis/src/registry.ts#L262) | 检查入口是否有 `Runtime` 记录。 |
| [`delete(plugin)`](../../../../vendor/cordis/src/registry.ts#L273) | 移除并返回 `Runtime`，触发该入口下所有 Fiber 的卸载。有记录时立即返回，无记录时返回 `undefined`。 |
| [`keys()`](../../../../vendor/cordis/src/registry.ts#L285)、[`values()`](../../../../vendor/cordis/src/registry.ts#L290)、[`entries()`](../../../../vendor/cordis/src/registry.ts#L295)、[`forEach(callback)`](../../../../vendor/cordis/src/registry.ts#L304) | 分别遍历入口函数、`Runtime`、`[callback, runtime]` 对，或以 `(runtime, callback)` 为参数访问每条记录。 |

卸载单次挂载并等待清理完成时，使用 `await fiber.dispose()`。[Fiber 清理](../../../../vendor/cordis/src/fiber.ts#L292)

从 profile 启动时，Loader 按 YAML 条目的 `name` 导入模块，再调用 `ctx.registry.plugin()`。YAML 的 `id` 标识配置条目，可以通过 `fiber.entry?.options.id` 读取。[Loader 挂载](../../../../vendor/loader/src/config/entry.ts#L274) · [关联配置条目](../../../../vendor/loader/src/index.ts#L151)

<a id="ctx-plugin"></a>
### `ctx.plugin()`

`ctx.plugin(plugin, config)` 转发到 `RegistryService.plugin(plugin, config?, getOuterStack = buildOuterStack())`。`plugin` 是插件定义，`config` 是这次挂载的原始配置，`getOuterStack` 是用来生成诊断调用栈的函数，默认由 `buildOuterStack()` 提供。[方法转发](../../../../vendor/cordis/src/reflect.ts#L233) · [挂载实现](../../../../vendor/cordis/src/registry.ts#L331)

1. **解析入口。** 函数或类直接作为 `callback`，对象插件使用它的 `apply` 函数。入口无效或当前 Fiber 已销毁时，调用会抛错。[入口解析](../../../../vendor/cordis/src/registry.ts#L237) · [挂载检查](../../../../vendor/cordis/src/registry.ts#L331)
2. **查找共享记录。** Registry 用 `callback` 的引用查找 `Runtime`。首次注册时创建记录，保存 `name`、`callback`、`fibers` 和 `Config`。[Runtime 创建](../../../../vendor/cordis/src/registry.ts#L337)
3. **创建本次挂载。** Registry 把当前 Context、原始配置、解析后的 `inject` 和共享的 `Runtime` 交给新的 Fiber。Fiber 等依赖就绪后执行插件入口。[Fiber 创建](../../../../vendor/cordis/src/registry.ts#L345) · [依赖驱动启动](../../../../vendor/cordis/src/fiber.ts#L639)

方法立即返回一个可 `await` 的 Fiber 句柄。`await ctx.plugin(...)` 等待当前生命周期工作结束，启动失败时会抛出错误。若依赖尚未就绪，等待也可能结束，此时 Fiber 仍处于 `PENDING`。[返回值](../../../../vendor/cordis/src/registry.ts#L345) · [Fiber 等待](../../../../vendor/cordis/src/fiber.ts#L776)

<a id="ctx-inject"></a>
### `ctx.inject()`

每次调用 `ctx.inject(deps, callback)` 都会挂载一个子插件，并为它创建一个子 Fiber。这个子插件用 `deps` 声明自己的服务依赖，用 `callback` 作为插件入口。通过 `RegistryService.inject()`，这次调用等价于 `ctx.plugin({ inject: deps, apply: callback, name: callback.name })`。普通依赖用服务名数组声明，需要附带拦截配置时，用以服务名为键的对象。[实现](../../../../vendor/cordis/src/registry.ts#L315) · [依赖声明类型](../../../../vendor/cordis/src/registry.ts#L18)

以 `tool-fs` 为例，模块导出的 `inject = ['tools', 'fs', 'systemPrompt', 'workingDirectory']` 适用于整个插件。Cordis 等这四项服务都就绪后才执行模块的 `apply()`。开启图片读取时，`apply()` 通过 `ctx.inject(['attachments'], (imageCtx) => applyReadImageTool(imageCtx))` 为 `read_image` 单独声明依赖。其他文件工具可以在 `attachments` 缺席时照常注册。[模块级依赖](../../../../packages/fs/tool-fs/src/index.ts#L24) · [图片工具注册](../../../../packages/fs/tool-fs/src/index.ts#L70)

所有依赖就绪后，Cordis 用子 Fiber 的 Context 调用回调。任一依赖消失时，Cordis 清理该子 Fiber 的注册和 effect。依赖恢复或被替换后，Cordis 在依赖全部就绪时重新运行回调。用回调收到的 Context 注册工具和监听器，并为需要显式释放的资源登记 `ctx.effect()` 清理，让它们随这项功能一起卸载。[依赖变更](../../../../vendor/cordis/src/fiber.ts#L652) · [卸载与重新激活](../../../../vendor/cordis/src/fiber.ts#L734)

`ctx.inject()` 立即返回一个可等待的子 Fiber。若依赖仍缺失，`await` 可以在 Fiber 处于 `PENDING` 时结束。需要主动卸载这项功能时，调用 `await fiber.dispose()`。[等待语义](#ctx-plugin) · [Fiber 清理](../../../../vendor/cordis/src/fiber.ts#L292)

- `tool-bash` 在 `jobs` 就绪后把前台版工具换成支持后台任务的版本。若 `jobs` 卸载时父插件仍在运行，则恢复前台版。[源码](../../../../packages/shell/tool-bash/src/index.ts#L588)
- `gateway` 用两次调用声明不同的依赖。RPC 拦截器只需要 `connection`，WebSocket 流的注册还需要 `webServer`。[源码](../../../../packages/api/gateway/src/index.ts#L255)

<a id="fiber"></a>
## Fiber

每次 `ctx.plugin(...)` 挂载都会创建一个 Fiber。构造时，`parent.extend({ fiber: this })` 创建插件自己的 `ctx`，它继承挂载方的上下文，并将 `ctx.fiber` 指向新 Fiber。 [上下文创建](../../../../vendor/cordis/src/fiber.ts#L270)

### 核心属性

| 属性 | 用途 |
|---|---|
| `uid` | 注册编号。根 Fiber 的编号为 `0`。普通 Fiber 开始销毁时会把编号设为 `null`。 |
| `parent` / `ctx` | 分别是发起挂载的上下文和交给插件的上下文。 |
| `runtime` | 共享的插件运行时记录。根 Fiber 的这个字段为 `null`。 |
| `inject` / `committed` / `provisions` | 分别记录所需服务、本次激活及清理期间保留的服务实现，以及这个 Fiber 提供的服务实现。 |
| `rawConfig` / `effectiveConfig` | 原始输入和最近一次启动或成功原地更新所用的解析后配置。直接设置 `rawConfig` 只保存输入。 |
| `state` / `inertia` | 当前生命周期状态和进行中的加载或卸载 Promise。 |

### 核心方法

| 方法 | 用途 |
|---|---|
| [`new Fiber(parent, config, inject, runtime, getOuterStack)`](../../../../vendor/cordis/src/fiber.ts#L256) | `runtime` 为 `null` 时初始化根 Fiber。挂载插件时，创建子上下文，将清理注册到父 Fiber，并检查初始依赖。 |
| [`effect(setup, label?)`](../../../../vendor/cordis/src/fiber.ts#L446) / [`getEffects()`](../../../../vendor/cordis/src/fiber.ts#L596) | 立即执行 `setup`，并由当前 Fiber 管理它交出的清理函数。可选的 `label` 用于诊断，`getEffects()` 返回已注册副作用的元数据。参见下方的 [`effect()` 调用方式](#fiber-effect)。 |
| [`await()`](../../../../vendor/cordis/src/fiber.ts#L776) | 等待当前生命周期工作并抛出启动错误。依赖缺失时，方法可能在 Fiber 仍为 `PENDING` 时返回。 |
| [`update(config)`](../../../../vendor/cordis/src/fiber.ts#L805) | 提交新配置。默认重启，更新 hook 可以保留当前实例。 |
| [`restart()`](../../../../vendor/cordis/src/fiber.ts#L790) | 使用已保存的配置请求重启。运行中的实例先卸载，再重新加载。 |
| [`dispose()`](../../../../vendor/cordis/src/fiber.ts#L292) | 永久移除普通 Fiber 并等待清理。根 Fiber 的 `dispose()` 委托给 `restart()`。 |
| [`resolveConfig(config)`](../../../../vendor/cordis/src/fiber.ts#L698) | 解析并校验候选配置。应用配置使用 `update()`。 |
| [`assertActive()`](../../../../vendor/cordis/src/fiber.ts#L379) | `uid` 为 `null` 时抛错。判断插件是否正在运行时，检查 `state === FiberState.ACTIVE`。 |

下面是 Fiber 内部的依赖检查和状态转换方法。公开的 `restart()` 递增 `_revision`，通过 `_refresh()` 请求重新加载。当前实例的清理和下一次启动分别由 `_unload()` 和 `_reload()` 执行。 [重启入口](../../../../vendor/cordis/src/fiber.ts#L790)

| 内部方法 | 用途 |
|---|---|
| [`_checkImpl(name)`](../../../../vendor/cordis/src/fiber.ts#L625) | 查找当前作用域中由 `ACTIVE` Fiber 提供的服务实现，并运行可选的 `check`。可用时写入 `_store`，否则删除对应记录。 |
| [`_refresh()`](../../../../vendor/cordis/src/fiber.ts#L639) | 用 `_revision` 和各个依赖实现的 `uid` 计算目标 `epoch`。缺少任一必需依赖时，目标为 `INACTIVE`。 |
| [`_setEpoch(epoch)`](../../../../vendor/cordis/src/fiber.ts#L652) | 接收新的生命周期目标。没有正在执行的转换时，从 `INACTIVE` 切换到有效目标会启动加载，其他目标变化会启动卸载。 |
| [`_reload()`](../../../../vendor/cordis/src/fiber.ts#L703) | 将当前可用的依赖保存到 `committed`，登记依赖关系。目标 `epoch` 仍一致时，调用链是 `_reload() → _execute(this._runner) → _runner.execute()`。`_runner.execute()` 先解析配置，再调用 `runtime.callback(ctx, config)`，类插件则用 `new runtime.callback(ctx, config)` 创建实例。插件入口可以在此时调用 `ctx.provide()`、`ctx.on()` 等完成注册。启动失败或目标变化时进入卸载流程。 [启动动作](../../../../vendor/cordis/src/fiber.ts#L272) |
| [`_execute(runner)`](../../../../vendor/cordis/src/fiber.ts#L384) | 执行插件入口或 effect 的启动函数，收集它返回或产出的清理函数。 |
| [`_unload()`](../../../../vendor/cordis/src/fiber.ts#L734) | 等待依赖本 Fiber 所提供服务的其他 Fiber 完成卸载，执行本 Fiber 的清理函数，再解除它对上游服务的依赖。清理完后根据最新的 `epoch` 决定是否重新加载。 |
| [`_getState()`](../../../../vendor/cordis/src/fiber.ts#L602) | 根据 `uid`、启动错误和 `epoch` 推导稳定状态，依次判断 `DISPOSED`、`FAILED`、`ACTIVE` 和 `PENDING`。 |
| [`_updateState(callback)`](../../../../vendor/cordis/src/fiber.ts#L609) | 写入回调指定的状态，或用 `_getState()` 推导状态。状态变化时发出 `internal/status`，进出 `ACTIVE` 时再通知服务依赖方。 |

[副作用注册](../../../../vendor/cordis/src/fiber.ts#L443) · [配置访问](../../../../vendor/cordis/src/fiber.ts#L670) · [生命周期方法](../../../../vendor/cordis/src/fiber.ts#L771)

<a id="fiber-effect"></a>
#### `effect()` 调用方式

插件通常在入口（例如 `apply(ctx)`）中调用 `ctx.effect()`，让资源随插件一起卸载。插件运行期间也可以调用。比如 `LoggerService.exporter()` 在每次注册接收器时调用 `ctx.effect()`，让注册归调用方的 Fiber 管理。插件入口或服务方法决定何时登记副作用，传给 `ctx.effect()` 的 `setup` 负责执行操作并交出清理函数。 [日志接收器注册](../../../../vendor/cordis/src/logger.ts#L235)

- **同步初始化。** 调用 `const dispose = ctx.effect(setup)`，其中 `setup` 直接返回清理函数。需要提前清理时调用 `await dispose()`。同步初始化也可以交出异步清理函数。 [同步重载](../../../../vendor/cordis/src/fiber.ts#L443) · [异步清理示例](../../../../packages/session/session-persistence-sqlite/src/sqlite.ts#L985)
- **异步初始化。** 调用 `const dispose = await ctx.effect(asyncSetup)`，等待初始化完成并取得清理函数。例如目录选择器在创建并等待两个插件后返回 `unmount`。之后调用 `await dispose()` 才执行清理。 [等待初始化](../../../../vendor/cordis/src/fiber.ts#L578) · [目录选择器示例](../../../../packages/host/directory-picker-auto/src/index.ts#L78)
- **一组清理动作。** 使用 `ctx.effect(function* () { … })`，每次 `yield` 交出一个清理函数。同一个 effect 的清理函数按逆序执行。例如 `command-compact` 先注销命令，再等待已经开始的调用结束。 [命令注册示例](../../../../packages/compaction/command-compact/src/index.ts#L98)

<a id="fiber-lifecycle"></a>
### 生命周期与清理

依赖齐备后，Fiber 从 `PENDING` 经 `LOADING` 进入 `ACTIVE`。依赖消失时，Fiber 经 `UNLOADING` 回到 `PENDING`。启动失败时，Cordis 先清理本次加载，再进入 `FAILED`。主动销毁普通 Fiber 最终进入 `DISPOSED`。 [状态转换](../../../../vendor/cordis/src/fiber.ts#L639)

子 Fiber 通过 `parent.fiber.effect(...)` 注册清理。比如 `workspace-controller` 用自己的 `ctx` 挂载 `DirectoryPickerController`，卸载父插件时会一并清理这个子插件。服务依赖还决定卸载顺序：服务提供方等待使用其服务的 Fiber 完成卸载，再清理自己的副作用。 [子 Fiber 清理注册](../../../../vendor/cordis/src/fiber.ts#L292) · [`DirectoryPickerController`](../../../../packages/api/workspace-controller/src/index.ts#L82) · [依赖卸载顺序](../../../../vendor/cordis/src/fiber.ts#L734)

<a id="fiber-stable-transitions"></a>
#### 稳定状态转换

普通插件 Fiber 从 `PENDING` 开始。`PENDING`、`ACTIVE`、`FAILED` 和 `DISPOSED` 是稳定状态。下表的起点和终点都是稳定状态，完整路径同时列出经过的 `LOADING` 和 `UNLOADING`。

| 起点 | 终点 | 触发条件 | 完整路径 | 函数与源码 |
|---|---|---|---|---|
| `PENDING` | `ACTIVE` | 依赖齐备，启动成功。 | `PENDING → LOADING → ACTIVE` | [`_refresh()` 计算目标](../../../../vendor/cordis/src/fiber.ts#L639) · [`_setEpoch()` 开始加载](../../../../vendor/cordis/src/fiber.ts#L660) · [`_reload()` 完成加载](../../../../vendor/cordis/src/fiber.ts#L724) |
| `PENDING` | `FAILED` | 首次启动失败，清理完成。 | `PENDING → LOADING → UNLOADING → FAILED` | [`_reload()` 记录错误并转入卸载](../../../../vendor/cordis/src/fiber.ts#L715) · [`_unload()` 完成清理](../../../../vendor/cordis/src/fiber.ts#L760) · [`_getState()` 判定失败](../../../../vendor/cordis/src/fiber.ts#L604) |
| `PENDING` | `PENDING` | 启动期间失去依赖，清理完成时仍缺依赖。 | `PENDING → LOADING → UNLOADING → PENDING` | [`_setEpoch()` 更新目标](../../../../vendor/cordis/src/fiber.ts#L657) · [`_reload()` 转入卸载](../../../../vendor/cordis/src/fiber.ts#L724) · [`_unload()` 完成清理](../../../../vendor/cordis/src/fiber.ts#L760) |
| `ACTIVE` | `PENDING` | 失去依赖，清理完成时仍缺依赖。 | `ACTIVE → UNLOADING → PENDING` | [`_refresh()` 算出 `INACTIVE`](../../../../vendor/cordis/src/fiber.ts#L639) · [`_setEpoch()` 开始卸载](../../../../vendor/cordis/src/fiber.ts#L663) · [`_unload()` 完成清理](../../../../vendor/cordis/src/fiber.ts#L760) |
| `ACTIVE` | `ACTIVE` | 重启，或依赖实现变化后重新加载成功。 | `ACTIVE → UNLOADING → LOADING → ACTIVE` | [`restart()` 递增 `_revision`](../../../../vendor/cordis/src/fiber.ts#L790) · [`_refresh()` 计算目标](../../../../vendor/cordis/src/fiber.ts#L639) · [`_setEpoch()` 开始卸载](../../../../vendor/cordis/src/fiber.ts#L663) · [`_unload()` 接续加载](../../../../vendor/cordis/src/fiber.ts#L760) · [`_reload()` 完成加载](../../../../vendor/cordis/src/fiber.ts#L724) |
| `ACTIVE` | `FAILED` | 重新加载失败，清理完成。 | `ACTIVE → UNLOADING → LOADING → UNLOADING → FAILED` | [`_unload()` 接续加载](../../../../vendor/cordis/src/fiber.ts#L760) · [`_reload()` 记录错误并转入卸载](../../../../vendor/cordis/src/fiber.ts#L715) · [`_getState()` 判定失败](../../../../vendor/cordis/src/fiber.ts#L604) |
| `FAILED` | `ACTIVE` | `update(config)` 清除错误，重试成功。 | `FAILED → LOADING → ACTIVE` | [`update()` 清除错误](../../../../vendor/cordis/src/fiber.ts#L812) · [`update()` 触发重试](../../../../vendor/cordis/src/fiber.ts#L827) · [`_reload()` 完成加载](../../../../vendor/cordis/src/fiber.ts#L724) |
| `FAILED` | `FAILED` | `update(config)` 触发重试，启动再次失败。 | `FAILED → LOADING → UNLOADING → FAILED` | [`update()` 触发重试](../../../../vendor/cordis/src/fiber.ts#L827) · [`_reload()` 记录新错误](../../../../vendor/cordis/src/fiber.ts#L715) · [`_getState()` 判定失败](../../../../vendor/cordis/src/fiber.ts#L604) |
| `FAILED` | `PENDING` | 重试已经开始，随后失去依赖，清理完成时仍缺依赖。 | `FAILED → LOADING → UNLOADING → PENDING` | [`update()` 触发重试](../../../../vendor/cordis/src/fiber.ts#L827) · [`_reload()` 转入卸载](../../../../vendor/cordis/src/fiber.ts#L724) · [`_unload()` 完成清理](../../../../vendor/cordis/src/fiber.ts#L760) |
| `PENDING` | `DISPOSED` | 等待依赖时被销毁。 | `PENDING → UNLOADING → DISPOSED` | [`dispose()` 直接开始清理](../../../../vendor/cordis/src/fiber.ts#L308) · [`_getState()` 检查 `uid`](../../../../vendor/cordis/src/fiber.ts#L603) |
| `ACTIVE` | `DISPOSED` | 运行中的插件被销毁。 | `ACTIVE → UNLOADING → DISPOSED` | [`dispose()` 清空 `uid`](../../../../vendor/cordis/src/fiber.ts#L294) · [`_setEpoch()` 开始卸载](../../../../vendor/cordis/src/fiber.ts#L663) · [`_getState()` 检查 `uid`](../../../../vendor/cordis/src/fiber.ts#L603) |
| `FAILED` | `DISPOSED` | 启动失败后被销毁。 | `FAILED → UNLOADING → DISPOSED` | [`dispose()` 直接开始清理](../../../../vendor/cordis/src/fiber.ts#L308) · [`_getState()` 检查 `uid`](../../../../vendor/cordis/src/fiber.ts#L603) |

加载或卸载期间如果目标 `epoch` 再次变化，可以重复经过 `LOADING → UNLOADING`，直到当前操作稳定下来。普通 Fiber 到达 `DISPOSED` 后生命周期结束。根 Fiber 从 `ACTIVE` 开始，其 `dispose()` 委托给 `restart()`。 [加载结束](../../../../vendor/cordis/src/fiber.ts#L724) · [清理结束](../../../../vendor/cordis/src/fiber.ts#L760) · [根 Fiber](../../../../vendor/cordis/src/fiber.ts#L348)

<a id="fiber-construction"></a>
#### 构造

普通 Fiber 在第一次检查依赖前，先创建插件上下文并登记清理。监听器收到 `internal/plugin` 时，可以补充依赖，也可以开始销毁这个 Fiber。 [构造函数](../../../../vendor/cordis/src/fiber.ts#L256)

1. **准备参数。** `ctx.plugin(plugin, config)` 由 Registry 解析插件入口、取得共享的 `runtime`，并将 `inject` 转为依赖表。发起挂载的 Context 作为 `parent` 传给 `new Fiber(...)`。通过 `ctx.inject()` 挂载的子插件也走这条路径。 [`RegistryService.inject()`](../../../../vendor/cordis/src/registry.ts#L315) · [创建 Fiber](../../../../vendor/cordis/src/registry.ts#L331)
2. **建立插件上下文。** 构造函数保存原始配置、分配 `uid`，并用 `parent.extend({ fiber: this })` 创建插件的 Context。此时 `ctx.fiber` 指向新 Fiber，`parent.fiber` 指向父 Fiber，新 Fiber 处于 `PENDING`。 [上下文创建](../../../../vendor/cordis/src/fiber.ts#L265)
3. **准备启动动作并登记清理。** `_runner.execute` 保存解析配置和调用插件入口的动作。构造函数通过 `parent.fiber.effect(...)` 将新 Fiber 加入 `runtime.fibers`，并为它设置 `dispose()`。 [启动动作](../../../../vendor/cordis/src/fiber.ts#L272) · [清理登记](../../../../vendor/cordis/src/fiber.ts#L292)
4. **发出 `internal/plugin`。** Loader 的同步监听器关联 `fiber.entry`，并把 YAML 条目里的 `inject` 合并到 `fiber.inject`。其他监听器此时可以调用 `fiber.dispose()`。如果 `uid` 已清空，或父 Fiber 正在卸载，构造函数就跳过首次依赖检查。 [发出事件](../../../../vendor/cordis/src/fiber.ts#L327) · [Loader 合并依赖](../../../../vendor/loader/src/index.ts#L149)
5. **检查依赖。** `_checkImpl(name)` 查找每项依赖的可用实现，`_refresh()` 根据依赖和重启版本号计算目标 `epoch`。依赖缺失时继续等待。依赖齐备时进入 `LOADING`，执行启动动作，成功后进入 `ACTIVE`。 [首次依赖检查](../../../../vendor/cordis/src/fiber.ts#L338) · [状态转换](../../../../vendor/cordis/src/fiber.ts#L639)

类插件的启动动作依次执行类构造函数、`initHooks` 中的回调和可选的 `[Service.init]()`。方法级 `@Inject` 在 `initHooks` 中通过 `ctx.inject()` 创建子 Fiber，等它自己的依赖就绪后再调用原方法。 [类插件启动](../../../../vendor/cordis/src/fiber.ts#L278) · [方法装饰器](../../../../vendor/cordis/src/registry.ts#L54)

`new Context()` 创建根 Fiber 时传入 `runtime = null`。构造函数直接使用根 Context，将 `uid` 设为 `0`，从 `ACTIVE` 开始。 [根 Fiber 分支](../../../../vendor/cordis/src/fiber.ts#L348)

<a id="fiber-dependency-resolution"></a>
#### 依赖解决

假设 B 的 Fiber 正在等待 A 提供的 `fs`。服务就绪后，`ReflectService.notify()` 会让 B 重新检查它能否启动。

1. **提供方通知服务就绪。** A 在启动时调用 `ctx.provide('fs', ...)`，Cordis 先登记实现，等 A 进入 `ACTIVE` 时再通知。如果 A 已处于 `ACTIVE`，`provide()` 会立即通知。 [提供方状态变化](../../../../vendor/cordis/src/fiber.ts#L609) · [服务注册](../../../../vendor/cordis/src/reflect.ts#L324)
2. **`notify()` 找到受影响的 Fiber。** 它遍历 Registry 中的 Fiber，找到 `inject` 声明了 `fs` 且隔离标签匹配的 Fiber，直接调用它们的 `_checkImpl('fs')` 和 `_refresh()`。 [服务通知](../../../../vendor/cordis/src/reflect.ts#L357)
3. **B 检查全部依赖。** `_checkImpl()` 确认提供方是 `ACTIVE`，并运行实现提供的可选 `check`。通过后，它将实现记入 B 的 `_store`。`_refresh()` 用这张表检查全部依赖。如果 `fs` 是最后一个缺项，目标 `epoch` 会从 `INACTIVE` 变为有效值，`_setEpoch()` 随即调用 `_reload()`，让 B 进入 `LOADING`。 [依赖检查与启动](../../../../vendor/cordis/src/fiber.ts#L625)

`notify()` 直接调用 Fiber 的依赖检查方法来触发启动。完成这一轮检查后，它再发出 `internal/service` 事件，通知监听器服务有变化。 [事件发出位置](../../../../vendor/cordis/src/reflect.ts#L371)

<a id="fiber-dependency-loss"></a>
#### 失去依赖

B 正在使用 A 提供的 `fs` 时，如果 `fs` 变得不可用，B 会卸载当前实例。清理期间，B 仍可以通过 `ctx.fs` 使用本次启动时绑定的实现。 [服务查找](../../../../vendor/cordis/src/reflect.ts#L147)

1. **服务失效。** A 离开 `ACTIVE` 时会通知依赖方重新检查。此时即使服务仍在注册表里，依赖检查也只接受由 `ACTIVE` Fiber 提供的实现。直接调用 `ctx.provide('fs', ...)` 返回的清理函数时，Cordis 会先从服务表移除该实现，再通知依赖方。 [状态变化通知](../../../../vendor/cordis/src/fiber.ts#L609) · [可用性检查](../../../../vendor/cordis/src/reflect.ts#L273) · [服务注销](../../../../vendor/cordis/src/reflect.ts#L333)
2. **B 进入 `UNLOADING`。** `notify()` 找到依赖 `fs` 且隔离标签匹配的 B，调用 `_checkImpl('fs')`。如果没有可用实现，B 会从 `_store` 删除这一项。`_refresh()` 将目标 `epoch` 设为 `INACTIVE`，`_setEpoch()` 随即启动卸载。其他 Fiber 如果依赖 B 提供的服务，也会在 B 离开 `ACTIVE` 时收到通知。 [查找受影响的 Fiber](../../../../vendor/cordis/src/reflect.ts#L357) · [刷新目标状态](../../../../vendor/cordis/src/fiber.ts#L625) · [通知下游依赖方](../../../../vendor/cordis/src/fiber.ts#L609)
3. **保留旧实现直到清理完成。** `_store` 保存下一次启动可用的实现，`committed` 保存本次启动绑定的实现。清理函数通过 `ctx.fs` 仍能读到 B 的 `committed.fs`。如果 A 也在卸载，它会等 B 解除依赖关系后才清理自己的副作用。B 清理完毕后，从旧实现的 `dependents` 中移除自己，并清空 `committed`。 [已绑定服务的查找](../../../../vendor/cordis/src/reflect.ts#L161) · [卸载与释放依赖](../../../../vendor/cordis/src/fiber.ts#L734)
4. **按最新依赖决定后续状态。** 如果仍缺依赖，B 进入 `PENDING`，保留原 Fiber 等待服务通知。如果清理期间依赖已恢复齐全，B 会在清理完成后直接进入 `LOADING`，用当前可用的实现重新启动。 [卸载后的状态判断](../../../../vendor/cordis/src/fiber.ts#L760)

<a id="reflect-service"></a>
## ReflectService

`ReflectService` 按作用域查找服务，并在 Context 上提供服务方法的调用入口。 [源码](../../../../vendor/cordis/src/reflect.ts#L244)

### 核心属性

| 属性 | 用途 |
|---|---|
| `store` | 以作用域标签为键保存已注册的服务实现。 |
| `props` | 保存暴露在 Context 上的属性定义。 |

### 核心方法

| 方法 | 用途 |
|---|---|
| [`new ReflectService(ctx)`](../../../../vendor/cordis/src/reflect.ts#L249) | 将反射服务关联到 Context，并安装反射、Fiber、插件注册和事件的内置转发入口。 |
| [`get(name)`](../../../../vendor/cordis/src/reflect.ts#L269) | 在调用者的作用域查找服务。 |
| [`provide(name, value?, check?)`](../../../../vendor/cordis/src/reflect.ts#L313) | 注册服务实现，并把清理交给调用者的 Fiber。 |
| [`accessor(name, options)`](../../../../vendor/cordis/src/reflect.ts#L385) / [`mixin(source, mixins)`](../../../../vendor/cordis/src/reflect.ts#L404) | 在 Context 上暴露访问器或指定的服务成员。 |

<a id="events-service"></a>
## EventsService

`EventsService` 负责注册和分发事件监听器。 [源码](../../../../vendor/cordis/src/events.ts#L130)

### 核心属性

| 属性 | 用途 |
|---|---|
| `_hooks` | 按事件名保存监听器的内部表。每条注册记录包含其 Context。 |

### 核心方法

| 方法 | 用途 |
|---|---|
| [`new EventsService(ctx)`](../../../../vendor/cordis/src/events.ts#L133) | 将事件服务关联到 Context，并注册内置的监听器处理、配置更新和配置解析处理器。 |
| [`on(name, listener)`](../../../../vendor/cordis/src/events.ts#L298) / [`once(name, listener)`](../../../../vendor/cordis/src/events.ts#L321) | 在当前 Fiber 下注册可重复触发或只触发一次的监听器。 |
| [`emit(...)`](../../../../vendor/cordis/src/events.ts#L207) / [`parallel(...)`](../../../../vendor/cordis/src/events.ts#L193) | `emit` 同步调用监听器。`parallel` 并行运行监听器并等待全部结束。 |
| [`bail(...)`](../../../../vendor/cordis/src/events.ts#L232) / [`serial(...)`](../../../../vendor/cordis/src/events.ts#L218) | 遇到首个 `undefined`、`null`、`false` 以外的返回值就停止。`serial` 会逐个等待监听器。 |
| [`waterfall(...)`](../../../../vendor/cordis/src/events.ts#L250) | 用监听器包裹最终回调。监听器调用 `next()` 后才会继续执行后续链条。 |

<a id="logger-service"></a>
## LoggerService

`LoggerService` 把结构化日志消息发送给已注册的接收器。 [源码](../../../../vendor/cordis/src/logger.ts#L191)

### 核心属性

| 属性 | 用途 |
|---|---|
| `buffer` / `bufferSize` | 最近的日志消息和缓冲区保留的最大条数。 |
| `exporters` | 已注册的日志接收器。 |

### 核心方法

| 方法 | 用途 |
|---|---|
| [`new LoggerService(ctx)`](../../../../vendor/cordis/src/logger.ts#L202) | 创建可调用的日志服务，并安装默认接收器，将最近的日志消息保存在缓冲区。 |
| [`ctx.logger(name?)`](../../../../vendor/cordis/src/logger.ts#L243) | 创建日志记录器。省略名称时，使用配置中的名称，或根据调用者的 Fiber 生成名称。 |
| [`error()`](../../../../vendor/cordis/src/logger.ts#L256) / [`warn()`](../../../../vendor/cordis/src/logger.ts#L256) / [`info()`](../../../../vendor/cordis/src/logger.ts#L256) / [`debug()`](../../../../vendor/cordis/src/logger.ts#L256) | 按对应级别记录日志。 |
| [`exporter(exporter)`](../../../../vendor/cordis/src/logger.ts#L235) | 注册日志接收器并返回清理函数。当前 Fiber 卸载时也会清理这条注册。 |
