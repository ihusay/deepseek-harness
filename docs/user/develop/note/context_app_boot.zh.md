# `boot()` 如何启动 Host

[English](context_app_boot.md) | 中文

## 摘要

从 profile 配置文件一路看到 Cordis 插件树开始运行。CLI 调用 `boot()` 时，已经选定 profile 并收集了它的补丁。[CLI 启动与 profile 解析](cli_startup.zh.md#profile)

## 目录

- [启动顺序](#startup-order)
- [启动失败与清理](#startup-failure)

<a id="startup-order"></a>
## 启动顺序

1. **创建 Host 上下文。** `new Context()` 创建 Cordis 根上下文和它的 Fiber。`boot()` 把 `ctx.baseUrl` 设为根 `cordis.yml` 所在目录，提供 `dshHomePath`，再等待 `ctx.plugin(Loader)` 安装 Loader。[`boot()` 实现](../../../../packages/boot/app-boot/src/index.ts#L1220)
2. **执行 `prepare(ctx)`。** 这个可选回调在 Loader 可用后、profile 条目开始挂载前运行。执行 `pnpm dsh web` 时，`runProfile()` 传入的回调会提供 `profileContext` 和启动环境，安装包解析器，并提供 `cmdlineArgs`。Web 启动插件之后可以注入 `cmdlineArgs`，读取 `--port` 等参数。[CLI 准备回调](../../../../apps/cli/src/profile-boot.ts#L336) · [Web 启动插件](../../../../packages/bundle/web-app/src/startup.ts#L91)
3. **挂载插件树。** `mountRootInclude()` 向 Loader 注册内置的 `cordis:include` 条目。Include 读取根 `cordis.yml`，按顺序应用补丁，再把得到的配置树交给 Loader。插件实例会在所需服务就绪后激活。[根 Include 配置](../../../../packages/boot/app-boot/src/index.ts#L758) · [应用补丁](../../../../vendor/include/src/index.ts#L449)
4. **等待并检查启动结果。** `boot()` 等 Loader 处理完当前启动工作，再调用 `auditStartupEntries()`。根 Include 或必需条目未能激活时，检查会抛出 `StartupError`。其他未激活的条目可以记录为警告，已启动的插件继续运行。检查通过后，`boot()` 返回根 `ctx`。如果应用在启动期间已经退出，`boot()` 确认 Loader 已释放后就返回上下文。[启动检查](../../../../packages/boot/app-boot/src/index.ts#L1173)

<a id="startup-failure"></a>
## 启动失败与清理

如果启动抛错，`boot()` 会等待 `ctx.fiber.dispose()` 清理已经启动的 Host 组件，再把错误交回 CLI。独立的 `diagnostics` 上下文共用 Host 的 logger，继续收集启动和清理期间的警告与错误。如果错误是 `StartupError`，`boot()` 会把收集到的消息附在错误上。成功和失败都会进入 `finally`，释放诊断上下文的 Fiber。[失败处理](../../../../packages/boot/app-boot/src/index.ts#L1257)
