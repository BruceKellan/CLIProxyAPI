# 04. SDK 嵌入与二次扩展

## 一句话结论

这个项目更准确的心智模型，不是“一个 CLI 程序 + 一些只给 CLI 自己复用的内部包”，而是“CLI 只是默认宿主，真正的公开运行时以 `sdk/cliproxy` 为嵌入面”。README 已经把它明确列为“Reusable Go SDK for embedding the proxy”，`docs/sdk-usage.md` 进一步说明：外部程序可以不依赖 CLI 二进制，直接嵌入 routing、authentication、hot-reload 和 translation 层（`README.md:61`；`docs/sdk-usage.md:3-4`）。

## 为什么它不是“CLI 程序 + 若干内部包”

### 1. CLI 入口本身就在调用公开 SDK

`internal/cmd/run.go` 的 `StartService` / `StartServiceBackground` 并没有自己手搓一套独立运行时，而是直接走：

- `cliproxy.NewBuilder()`
- `WithConfig(...)`
- `WithConfigPath(...)`
- 可选 `WithLocalManagementPassword(...)`
- `Build()`
- `service.Run(...)`

这说明 CLI 只是 `sdk/cliproxy` 的一个宿主，而不是唯一实现（`internal/cmd/run.go:27-55, 60-79`）。

### 2. Builder 暴露的是标准 SDK 装配面，不是 CLI 私有辅助代码

`Builder` 公开了大量宿主可注入的扩展点：

- token / API key client provider
- watcher factory
- lifecycle hooks
- request access manager
- core auth manager
- server options
- local management password
- post-auth hook

如果它只是“CLI + 内部包”，通常不会把这些边界做成链式、可替换、可嵌入的公开 API（`sdk/cliproxy/builder.go:18-165`）。

### 3. Service 负责的是完整运行时，而不是 CLI 私有启动脚本

`Service.Run()` 会统一处理：

- `api.Server` 创建与启动
- websocket route 挂载与认证变更处理
- `OnBeforeStart` / `OnAfterStart` hooks
- model registry 刷新回调
- config/auth watcher
- reload 后 selector、config、server client、executor 重新绑定
- core auth auto-refresh

这些都属于可复用 runtime 的职责，而不是某个 CLI 命令的临时逻辑（`sdk/cliproxy/service.go:523-688`）。

### 4. 文档与示例也在把它定义成“宿主 + SDK”结构

项目没有只给 CLI 写使用说明，而是专门提供了：

- `docs/sdk-usage.md`：讲怎么把 proxy 当 Go SDK 嵌入
- `docs/sdk-advanced.md`：讲怎么扩展 executor / translator / model registry
- `examples/custom-provider`
- `examples/translator`
- `examples/http-request`

这是一条非常完整的“嵌入 -> 扩展 -> 示例验证”路径，本质上就是 SDK 项目的组织方式（`docs/sdk-usage.md:1-164`；`docs/sdk-advanced.md:1-139`）。

## 分层理解

可以把这个项目理解为下面这条分层链路：

```text
CLI / 你的 Go 宿主
  -> cliproxy.Builder
    -> cliproxy.Service
      -> api.Server
      -> access.Manager            # 入站请求鉴权
      -> core auth.Manager         # 出站 provider 凭据选择、刷新、执行
        -> executor                # 每个 provider 的真正调用实现
        -> translator              # 协议格式互转
      -> model registry            # /v1/models 暴露与模型可见性
      -> watcher / auto-refresh / lifecycle hooks
```

### Builder：装配边界

`Builder` 的职责是把“宿主希望怎么跑”装配成一个可运行的 `Service`。

关键注入点包括：

- `WithConfig` / `WithConfigPath`：最基本的运行配置；`Build()` 缺任一项都会报错（`sdk/cliproxy/builder.go:75-97, 166-173`）
- `WithTokenClientProvider` / `WithAPIKeyClientProvider`：替换凭据来源（`sdk/cliproxy/builder.go:99-109`）
- `WithWatcherFactory`：替换热更新 watcher（`sdk/cliproxy/builder.go:111-115`）
- `WithHooks`：接入启动生命周期（`sdk/cliproxy/builder.go:117-121`）
- `WithAuthManager`：替换 legacy token lifecycle manager（`sdk/cliproxy/builder.go:123-127`）
- `WithRequestAccessManager`：替换入站 `access` 认证链（`sdk/cliproxy/builder.go:129-133`）
- `WithCoreAuthManager`：替换出站 provider auth 与执行管理器（`sdk/cliproxy/builder.go:135-139`）
- `WithServerOptions` / `WithLocalManagementPassword` / `WithPostAuthHook`：扩展 HTTP server 行为（`sdk/cliproxy/builder.go:141-164`）

`Build()` 本身还会补齐默认实现：默认 token/API key provider、默认 watcher、默认 access manager、根据路由策略选择 selector，并把 default RoundTripper provider、config、OAuth model alias 绑定到 core manager（`sdk/cliproxy/builder.go:175-239`）。

### Service：真正的运行时

`Service` 才是长生命周期 runtime。

它在 `Run()` 里统一完成：

- 创建 `api.Server`，把 `coreManager`、`accessManager`、`configPath`、`serverOptions` 注进去（`sdk/cliproxy/service.go:523-525`）
- 挂 websocket 路由，并在 websocket auth 变化时重置现有连接（`sdk/cliproxy/service.go:530-549`）
- 启动前后执行 hooks（`sdk/cliproxy/service.go:551-553, 607-609`）
- 注册模型刷新回调（`sdk/cliproxy/service.go:555-590`）
- 启动 config/auth watcher，在 reload 时更新 selector、server clients、core config，并重新绑定 executors（`sdk/cliproxy/service.go:611-680`）
- 启动 core auth auto-refresh（`sdk/cliproxy/service.go:684-688`）

`sdk-usage` 对这个角色的描述也很明确：service 会管理 config/auth watching、background token refresh 和 graceful shutdown；取消 context 即可停服务（`docs/sdk-usage.md:36-45, 149-157, 161-164`）。

### access：入站请求谁能进来

`access` 层解决的是“谁可以调用这个代理服务”。

在 `Builder.Build()` 里，配置先注册到 access provider，再把已注册 providers 的快照塞到 `accessManager`（`sdk/cliproxy/builder.go:195-202`）。`Service.Run()` 再把这个 manager 传给 `api.Server`（`sdk/cliproxy/service.go:523-525`）。

因此，`access` 面向的是：

- 进入 proxy 的请求门禁
- 调用方身份识别
- 下游访问控制

它和上游 provider 的 OAuth/API key 是否可用，不是同一层事情。

### auth：出站 provider 凭据、选择、刷新与执行

这里说的 `auth`，核心是 `core auth manager` 这一层。

`Builder.Build()` 默认会创建一个 `coreauth.Manager`，并根据 routing strategy 选择 `RoundRobinSelector` 或 `FillFirstSelector`；随后再绑定 RoundTripper provider、config 和 OAuth model alias（`sdk/cliproxy/builder.go:203-227`）。

`Service.Run()` 则在运行时：

- 根据新配置更新 selector（`sdk/cliproxy/service.go:629-649`）
- 把最新 config / OAuth model alias 回写给 core manager（`sdk/cliproxy/service.go:656-662`）
- 开启 auto-refresh（`sdk/cliproxy/service.go:684-688`）

`sdk-usage` 也直接把它定义为 selection、execution、auto-refresh 的核心入口，并允许宿主替换自己的 manager，以自定义 transport 或 hooks（`docs/sdk-usage.md:79-116`）。

这里最容易混淆的是两类“认证”：

- `access`：入站，请求能不能进 proxy
- `core auth`：出站，proxy 用哪个 provider 凭据去打上游

### translator：协议/格式转换层

`translator` 层解决的是“客户端说一种 schema，上游 provider 要另一种 schema”的问题。

`docs/sdk-advanced.md` 把它定义成 translator registry：

- 处理 request transform
- 处理 response transform
- response 还要分别覆盖 stream / non-stream
- 内建支持 OpenAI / Gemini / Claude / Codex 等格式
- 也允许你注册新格式（`docs/sdk-advanced.md:10-15, 70-110`）

`examples/translator/main.go` 证明这层可以脱离 server 独立使用：

- 先检查 transform 是否存在
- 把 OpenAI request 转成 Gemini
- 再把 Gemini response 转回 OpenAI（`examples/translator/main.go:12-41`）

所以 translator 不是 handler 内部的零散 helper，而是一层独立的 schema conversion 能力。

### executor：真正把请求发到上游 provider

`executor` 层是每个 provider 的出站实现。

`sdk-advanced` 要求它实现 `auth.ProviderExecutor`，并可选实现请求预处理、stream、refresh 等能力（`docs/sdk-advanced.md:10-12, 16-68`）。

`examples/custom-provider/main.go` 里的 `MyExecutor` 是最直观的证据：

- `Identifier()` 返回 provider key（`examples/custom-provider/main.go:69-70`）
- `PrepareRequest()` 注入认证头（`examples/custom-provider/main.go:72-92`）
- `Execute()` 发起非流式调用（`examples/custom-provider/main.go:115-141`）
- `HttpRequest()` 支持 raw HTTP request path（`examples/custom-provider/main.go:143-156`）
- `ExecuteStream()` / `Refresh()` 覆盖流式与刷新行为（`examples/custom-provider/main.go:162-173`）

`examples/http-request/main.go` 又进一步展示了 executor/core manager 的 raw HTTP 复用路径：

- 先 `NewHttpRequest()`，再交给自己的 `http.Client`
- 或直接 `core.HttpRequest()`，让 manager 自动注入并发起请求（`examples/http-request/main.go:90-139`）

### model registry：对外公布哪些模型可见

model registry 的职责是“让哪些模型出现在 `/v1/models` 里”。

`docs/sdk-advanced.md` 明确要求：按 `auth ID + provider` 向全局 registry 注册模型，就能把它们暴露出去（`docs/sdk-advanced.md:112-124`）。

`examples/custom-provider/main.go` 展示了一个常见接法：在 `OnAfterStart` hook 里遍历已加载 auth，把自定义 provider 的模型注册到 `GlobalModelRegistry()`（`examples/custom-provider/main.go:188-197`）。

与此同时，`Service.Run()` 还会在模型目录变化时，按 provider 维度重新为受影响 auth 注册模型（`sdk/cliproxy/service.go:555-590`）。

这说明 model registry 是一层独立的“发布与可见性”能力：它依赖 auth 状态，但不等于 executor 本身。

## docs 与 examples 的关系

可以把官方文档和示例看成一条从“先嵌入”到“再扩展”的递进链路。

| 材料 | 回答的问题 | 适合什么时候看 |
| --- | --- | --- |
| `README.md` | 项目是否真的支持 SDK 嵌入 | 第一次判断项目定位时 |
| `docs/sdk-usage.md` | 怎样把 proxy 当成 Go SDK 嵌进自己的程序 | 先跑通最小 embed 时 |
| `docs/sdk-advanced.md` | 怎样自定义 provider、translator、模型注册 | 需要做二次开发时 |
| `examples/custom-provider` | 自定义 provider 的端到端最小闭环是什么 | 要自己接新上游 provider 时 |
| `examples/translator` | translator API 单独怎么用、怎么验证 | 要做 schema 互转或排查转换时 |
| `examples/http-request` | 不走标准 chat/stream handler 时，怎样复用 raw HTTP 路径 | 要直连自定义 HTTP 接口时 |

更具体地说：

- `docs/sdk-usage.md` 负责“宿主视角”：如何 `NewBuilder() -> Build() -> Run()`，以及如何挂 server options、hooks、custom client sources（`docs/sdk-usage.md:24-45, 46-71, 118-147`）
- `docs/sdk-advanced.md` 负责“扩展视角”：如何实现 executor、注册 translator、注册模型（`docs/sdk-advanced.md:16-124`）
- `examples/custom-provider` 把 advanced 文档里的几个扩展点串成了一个完整宿主（`examples/custom-provider/main.go:175-225`）
- `examples/translator` 刻意不拉起 server，只演示格式转换本身，方便隔离验证 translator 行为（`examples/translator/main.go:12-41`）
- `examples/http-request` 展示了另一个常被忽略的入口：即使不走完整 handler 流程，core auth manager 仍可复用凭据注入与请求执行能力（`examples/http-request/main.go:90-139`）

## 二次开发者从哪里扩展

### 1. 扩展 provider

入口：

- 实现 `auth.ProviderExecutor`
- 在自己的 `coreauth.Manager` 上 `RegisterExecutor(...)`
- 通过 `WithCoreAuthManager(core)` 交给 `Builder`

对应证据：

- `docs/sdk-advanced.md:16-68`
- `examples/custom-provider/main.go:175-203`

适用场景：

- 接入新的上游 AI provider
- 自定义 transport / 代理 / 认证头注入
- 支持 raw HTTP request execution

### 2. 扩展 translator

入口：

- 在 `sdk/translator` 默认 registry 里注册 request / response transforms
- 注意 response 要分别处理 stream 和 non-stream

对应证据：

- `docs/sdk-advanced.md:70-110`
- `examples/translator/main.go:12-41`

适用场景：

- 客户端协议和上游 provider 协议不一致
- 需要新增 chat/tool-calls/thinking 等 payload 格式映射

### 3. 扩展 model 注册

入口：

- 调 `cliproxy.GlobalModelRegistry().RegisterClient(authID, provider, models)`
- 常见时机是 service 启动后、auth 已加载完成之后

对应证据：

- `docs/sdk-advanced.md:112-124`
- `examples/custom-provider/main.go:188-197`
- `sdk/cliproxy/service.go:555-590`

适用场景：

- 让自定义 provider 的模型出现在 `/v1/models`
- 随 auth 或远端模型目录变化动态更新模型可见性

### 4. 扩展 server options

入口：

- `Builder.WithServerOptions(opts ...api.ServerOption)`
- 或使用 `WithLocalManagementPassword` / `WithPostAuthHook` 这类专门包装

对应证据：

- `sdk/cliproxy/builder.go:141-164`
- `docs/sdk-usage.md:46-71`
- `internal/cmd/run.go:37-44`
- `examples/custom-provider/main.go:204-210`

你可以在这里加：

- middleware
- gin engine configurator
- 自定义路由
- request logger factory
- keep-alive / local management 行为
- post-auth 持久化前钩子

### 5. 扩展 hooks

入口：

- `Builder.WithHooks(cliproxy.Hooks{OnBeforeStart, OnAfterStart})`

对应证据：

- `sdk/cliproxy/builder.go:53-64, 117-121`
- `sdk/cliproxy/service.go:551-553, 607-609`
- `docs/sdk-usage.md:137-147`
- `examples/custom-provider/main.go:188-212`

适用场景：

- 在 server 启动前调整配置
- 在服务 ready 后注册模型、挂宿主侧监控或状态上报
- 做启动期扩展，而不是请求期扩展

## 推荐二开阅读路径

1. 先看 `README.md:61`，确认项目公开定位就是“可嵌入 SDK”。
2. 通读 `docs/sdk-usage.md`，先建立 `Builder -> Service -> Run` 的最小宿主心智模型。
3. 看 `internal/cmd/run.go:19-84`，理解官方 CLI 自己也只是这个 SDK 的一个宿主。
4. 看 `sdk/cliproxy/builder.go:18-241`，记住有哪些正式注入点与默认装配逻辑。
5. 看 `sdk/cliproxy/service.go:523-688`，理解 runtime 真正帮你接管了哪些事：server、hooks、watcher、reload、auto-refresh、model refresh。
6. 再看 `docs/sdk-advanced.md`，把“怎么嵌入”切换成“怎么扩展”。
7. 最后按需求读示例：
   - 接新 provider：`examples/custom-provider/main.go:65-212`
   - 调 schema 转换：`examples/translator/main.go:12-41`
   - 走原始 HTTP：`examples/http-request/main.go:90-139`

## 容易踩坑的边界

1. `Build()` 不是只给一个 `cfg` 就够了，`configPath` 也是必填；少任一项都会直接报错（`sdk/cliproxy/builder.go:166-173`）。
2. 不要把 `access` 和 `auth` 混为一谈：
   - `access` 管入站调用者；
   - `core auth` 管出站 provider 凭据、选择、执行和刷新。
   改错层，问题不会消失（`sdk/cliproxy/builder.go:129-139, 195-227`；`sdk/cliproxy/service.go:523-525, 684-688`）。
3. `Builder` 里同时有 `authManager` 和 `coreManager` 两类 auth 相关对象；二开时真正决定 provider 执行、selector、transport、auto-refresh 的通常是 `coreManager`，不要把 `WithAuthManager` 和 `WithCoreAuthManager` 当成一回事（`sdk/cliproxy/builder.go:40-47, 123-139, 203-227`）。
4. 自定义 provider 只注册 executor 还不够；如果希望 `/v1/models` 正常暴露，还要补 model registry（`docs/sdk-advanced.md:112-124`；`examples/custom-provider/main.go:188-197`）。
5. translator 是有方向的。只做 request transform，不做 response transform，链路通常不完整；response 里还要区分 stream 和 non-stream（`docs/sdk-advanced.md:74-110`）。
6. hooks 是生命周期钩子，不是请求中间件。要改每个请求的行为，更适合走 `WithServerOptions` 或 executor/translator；要改启动期行为，再用 `OnBeforeStart` / `OnAfterStart`（`sdk/cliproxy/builder.go:53-64, 141-164`；`sdk/cliproxy/service.go:551-553, 607-609`）。
7. SDK 宿主和 CLI 不是两套完全独立的行为。`docs/sdk-usage.md` 明说 server options “mirror the internals used by the CLI server”，`internal/cmd/run.go` 也证明确实如此；embed 场景大多应该优先复用 Builder/Service，而不是绕开它们自己拼 runtime（`docs/sdk-usage.md:46-71`；`internal/cmd/run.go:27-55`）。
8. 原始 HTTP 场景不要重复造凭据注入逻辑。优先复用 core manager 的 `NewHttpRequest()` / `HttpRequest()` 路径，避免手写一套与 provider auth 脱节的 HTTP client（`examples/http-request/main.go:90-139`）。
