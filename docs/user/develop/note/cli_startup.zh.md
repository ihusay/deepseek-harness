# `pnpm dsh web` 的启动流程

[English](cli_startup.md) | 中文

## 摘要

沿着 `pnpm dsh web` 的源码，从命令行参数解析看到 React 首次渲染。想确认当前 checkout 会使用哪些插件和配置，可以在仓库根目录运行下面的命令。它会打印合并后的 profile，并在加载其中的插件之前退出。

```sh
pnpm dsh web --dump-config
```

## 目录

- [选择并组装 profile](#profile)
- [启动 Host 并提供页面](#host)
- [在浏览器加载并渲染应用](#browser)

<a id="profile"></a>
## 选择并组装 profile

1. 根目录的 [`dsh` 脚本](../../../../package.json) 执行 `node --import tsx/esm apps/cli/src/bin.ts web`。[`parseDshArgs()`](../../../../apps/cli/src/args.ts) 将 `web` 展开成 `--profile web`，然后 [`runCli()`](../../../../apps/cli/src/bin.ts) 把选中的 profile 和剩余参数交给 `runProfile()`。
2. [`runProfile()`](../../../../apps/cli/src/profile-boot.ts) 准备 profile 和包解析规则，再把合并后的补丁传给 `boot()`。内置 Web profile 依次应用 [`dsh-base`](../../../../packages/bundle/base/cordis.patch.yml) 和 [`dsh-web-app`](../../../../packages/bundle/web-app/cordis.patch.yml)。本地 profile 和命令行补丁可以改变最终的插件树。[Profile 与补丁参考](../../../../apps/cli/reference/README.zh.md)
3. [`boot()`](../../../../packages/boot/app-boot/src/index.ts#L1220) 创建 Host 的 Cordis 根 `Context` 并安装 Loader。CLI 先提供 `cmdlineArgs` 等启动信息，然后 Loader 加载 profile 的插件树。

### Profile 的具体解析流程

1. `composeProfile()` 调用 `prepareProfile('web')`，再由 `loadProfile()` 找到 `$DSH_HOME/profiles/web/`。如果目录里还没有 `package.json`，`loadProfile()` 会用 `PROFILE_TEMPLATES.web` 初始化它。默认生成的 `dsh.profile.bundles` 列表先是 `@deepseek-ai/dsh-base`，再是 `@deepseek-ai/dsh-web-app`。[CLI 准备 profile](../../../../apps/cli/src/profile-boot.ts) · [加载 profile](../../../../packages/boot/app-boot/src/profile.ts) · [内置模板](../../../../packages/util/official-bundles/src/bundles.ts)
2. `loadProfileDirectory()` 按顺序处理列表，先在 dsh 安装目录找 bundle，再到 profile 目录查找，然后读取 bundle 的 `dsh.bundle.patch` 字段指定的 YAML 文件。比如，`dsh-base` 的补丁用 `insert` 加入 `llm`、`session` 等条目，`dsh-web-app` 的补丁会修改部分基础条目，再加入 `webserver` 等 Web 条目。`loadOverlayPatches()` 读取文件，`parsePatchList()` 将每份文件解析成 YAML 数组。[基础 bundle 声明](../../../../packages/bundle/base/package.json) · [基础补丁](../../../../packages/bundle/base/cordis.patch.yml) · [Web 补丁](../../../../packages/bundle/web-app/cordis.patch.yml) · [YAML 解析](../../../../packages/boot/app-boot/src/index.ts)
3. `composeProfile()` 生成包解析表，并读取 `--patch` 指定的文件。`runProfile()` 在调用 `boot()` 前用 `readProfilePatches()` 收集补丁。应用顺序是 bundle 补丁、`$DSH_HOME/profiles/web/cordis.patch.yml`、`$DSH_HOME/cordis.patch.yml`，最后是按命令行顺序传入的 `--patch` 文件。如果遥测开关生成了补丁，这份补丁排在所有配置层之后。[补丁排序](../../../../packages/boot/app-boot/src/profile-context.ts)
4. `prepareProfile()` 写入空的 `cordis.yml`，作为 profile 的配置入口。启动时，`mountRootInclude()` 把这个文件和排好顺序的补丁交给 `cordis:include`。Include 在空列表上应用补丁，生成 Loader 要加载的插件条目。[根 Include 配置](../../../../packages/boot/app-boot/src/index.ts) · [应用补丁](../../../../vendor/include/src/index.ts)
5. 运行 `pnpm dsh web --dump-config`，在插件挂载前查看合并后的 profile。要查看额外补丁对结果的影响，可在同一条命令中传入 `--patch FILE`。

<a id="host"></a>
## 启动 Host 并提供页面

1. [`web-startup`](../../../../packages/bundle/web-app/src/startup.ts) 读取 `cmdlineArgs`，将解析后的 Web 参数注册成 `webStartup` 服务。[Web bundle 配置](../../../../packages/bundle/web-app/cordis.patch.yml) 使用这些值来配置 HTTP 监听器和其他 Web 插件。
2. [`WebServer`](../../../../packages/host/webserver/src/index.ts) 通过 `super(ctx, 'webServer')` 把自己注册为 `ctx.webServer`，并在初始化时调用 `server.listen()`。默认监听地址是 `127.0.0.1:3080`。
3. [`web-runtime` 插件](../../../../packages/bundle/web-app/src/index.ts) 注入 `webServer`，找到构建好的 `dist/index.html`，再通过 `ctx.plugin()` 加载 `frontend-static`。[`frontend-static`](../../../../packages/host/frontend-static/src/index.ts) 等 `webServer` 和 `connection` 就绪后，通过 `ctx.effect()` 注册静态文件的 fallback。插件卸载时，effect 会移除这个 fallback。
4. 必需插件就绪后，`web-runtime` 打印带启动 token 的 URL，并按配置打开默认浏览器。[Web 启动与打开浏览器](../../../../apps/cli/reference/README.zh.md#web-profile)

<a id="browser"></a>
## 在浏览器加载并渲染应用

1. 浏览器打开带 token 的 URL。认证流程用 token 换取签名 cookie，然后重定向到去掉 token 的 URL。fallback 读取 `dist/index.html`，通过 `renderIndex()` 注入 Host 提供的启动数据，再返回页面。[Web 认证流程](../../../../packages/bundle/web-app/README.zh.md) · [静态文件处理](../../../../packages/host/frontend-static/src/index.ts)
2. 构建后页面里的模块脚本执行 [`apps/web/src/main.ts`](../../../../apps/web/src/main.ts) 对应的代码。它找到 `#root`，创建 `AppWebEntry`，然后调用 `entry.run()`。[HTML 入口](../../../../apps/web/index.html)
3. [`AppWebEntry.run()`](../../../../packages/client/web/src/boot.ts) 等启动数据就绪，创建浏览器侧的 Cordis `Context` 并激活客户端插件，再把挂载点交给 [UI 渲染器](../../../../packages/client/ui-renderer/src/client/index.ts)。渲染器调用 React 的 `createRoot()` 和 `root.render()`。
