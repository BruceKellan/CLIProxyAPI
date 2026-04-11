# 00. 项目地图：先知道自己在读什么

这份文档先回答一个问题：`CLIProxyAPI` 到底是哪一类项目。

结论先说：它不是只有一层 HTTP 转发逻辑的“OpenAI 兼容代理”，而是把“北向协议入口”“南向 provider / 账号支持”“多账号运行时路由”“可嵌入 Go SDK”放在同一个仓库里。README 的产品面已经把这几层分开暗示出来了：对外主要暴露 OpenAI / Gemini / Claude 兼容接口，同时支持 Codex、Claude、Gemini、Qwen、iFlow 等账号接入与 OAuth 登录，还集成 Amp CLI 和可复用的 Go SDK。（README.md:40-89）

`go.mod` 进一步暴露了真实体量：`gin` 说明这里有完整 HTTP 服务层，`bubbletea`/`bubbles`/`lipgloss` 说明 TUI 不是附属脚本而是一等功能，`fsnotify` 说明有热更新，`go-git`/`pgx`/`minio` 说明认证与配置不只支持本地文件，`oauth2`/`websocket` 说明登录与长连接能力内建在项目里。（go.mod:0-31）

## 1. 项目定位：它不是单一用途的代理

可以把 CLIProxyAPI 理解成四层叠在一起：

1. **北向协议入口层**：对外暴露 OpenAI / Gemini / Claude 风格 API，以及 Amp 集成入口；这里解决的是“客户端如何接入这个系统”。（README.md:40-49；README.md:71-81）
2. **南向 provider / 账号支持层**：系统内部支持 Claude / Gemini / Codex / Qwen / iFlow / OpenAI-compatible 等多种账号和 provider 类型；这里解决的是“代理最终拿什么身份去调用上游”。（README.md:43-60；internal/config/config.go:87-126）
3. **运行时路由层**：支持多账户轮询、模型路由、重试与可用性切换，不是“一个 key 对一个 upstream”的简单代理。（README.md:51-60；sdk/cliproxy/builder.go:210-223）
4. **嵌入式 SDK 层**：仓库不是只给 `cmd/server` 自己用；README 直接暴露了 SDK 文档入口，而 CLI 启动本身也是通过 `sdk/cliproxy` 的 `Builder` / `Service` 完成装配的。（README.md:61-89；internal/cmd/run.go:27-55；sdk/cliproxy/builder.go:166-241）

这里最容易混淆的一点是：**Codex 更适合被理解成系统支持的一类 provider / auth 能力，而不是和 OpenAI / Gemini / Claude 并列的一层北向 API surface。** 读后续文档时，建议始终把“入口协议”和“上游 provider 类型”分开看。

如果你第一次读源码，最容易踩的坑是：一上来就从 `internal/api` 或某个 provider handler 开始读。这个仓库更适合按“入口 -> 装配 -> 运行时 -> 存储/热更新”去理解，因为真正决定系统形状的，不是单个 handler，而是 `cmd/server/main.go` 的模式分派、`sdk/cliproxy/builder.go` 的依赖装配、`sdk/cliproxy/service.go` 的生命周期，以及 `internal/store` / `internal/watcher` 的后端与热更新机制。（cmd/server/main.go:53-98；sdk/cliproxy/builder.go:18-241；sdk/cliproxy/service.go:29-92）

## 2. 仓库结构：按理解价值分层，而不是按目录名死记

| 路径 | 主要职责 | 读它的收益 |
| --- | --- | --- |
| `cmd/server/` | 进程入口、flag 解析、模式分支、后端选择 | 先搞清楚程序“怎么启动、能启动成什么样”。核心证据是 `cmd/server/main.go:53-98`、`cmd/server/main.go:161-418`、`cmd/server/main.go:461-581`。 |
| `internal/cmd/` | CLI 对 SDK 的薄封装 | 用来确认 CLI 最终如何落到 `cliproxy.Service`，以及前台/后台运行差异。见 `internal/cmd/run.go:19-98`。 |
| `sdk/cliproxy/` | 对外暴露的嵌入式运行时接口 | 这里定义了最关键的两个抽象：`Builder` 和 `Service`。见 `sdk/cliproxy/builder.go:18-241`、`sdk/cliproxy/service.go:29-92`、`sdk/cliproxy/service.go:466-769`。 |
| `internal/config/` | 配置结构、默认值、加载与 sanitize | 这是“配置到底长什么样、cloud 模式为什么允许空配置”的答案。见 `internal/config/config.go:26-129`、`internal/config/config.go:541-655`。 |
| `sdk/auth/` + `internal/store/` | 本地文件、Git、Postgres、Object Store 等凭据持久化 | 这是“auth/config 到底存在哪”的核心。见 `sdk/auth/filestore.go:20-170`、`internal/store/gitstore.go:26-212`、`internal/store/postgresstore.go:27-173`、`internal/store/objectstore.go:29-155`。 |
| `internal/watcher/` | config/auth 变更监听、热更新、增量 auth 更新 | 这是“为什么改配置/改 auth 文件不必重启”的答案。见 `internal/watcher/watcher.go:30-61`、`internal/watcher/config_reload.go:28-135`。 |
| `internal/managementasset/` + `internal/registry/` | 管理面静态资源更新、模型目录更新 | 这是后台 goroutine 的主要来源之一。见 `internal/managementasset/updater.go:57-111`、`internal/registry/model_updater.go:73-138`。 |

一个非常实用的阅读心法是：

- **把 `cmd/server/` 当成“调度层”看**，不要把它误认为核心业务层；它主要负责分支与前置条件。（cmd/server/main.go:53-98；cmd/server/main.go:461-581）
- **把 `sdk/cliproxy/` 当成“运行体”看**；CLI 入口和外部嵌入方都通过它来组织服务。（internal/cmd/run.go:27-55；sdk/cliproxy/builder.go:166-241；sdk/cliproxy/service.go:475-698）
- **把 `internal/store/`、`internal/watcher/`、`internal/registry/` 当成 runtime 支撑层看**；很多“为什么会这样”的答案在这里，而不在入口文件里。（cmd/server/main.go:236-391；internal/watcher/config_reload.go:79-135；internal/registry/model_updater.go:101-138）

## 3. 全仓最关键的几个抽象

### 3.1 `Config`：不只是“端口 + key”

`internal/config/config.go` 里的 `Config` 远不止一个服务器配置对象。它同时承载：网络监听（`Host` / `Port` / `TLS`）、管理面开关（`RemoteManagement`）、认证目录（`AuthDir`）、运行时行为（`RequestRetry`、`Routing`、`WebsocketAuth`）、多类 provider 配置（Gemini/Codex/Claude/OpenAI-compat/Vertex-compat）、Amp CLI 设置、OAuth model alias、payload 规则等。（internal/config/config.go:26-129；internal/config/config.go:154-259；internal/config/config.go:273-310）

所以读这个项目时，不要把 config 理解成“读取 YAML 后塞给 server”的简单对象；它实际上是多个子系统共享的 runtime contract。（internal/config/config.go:26-129；sdk/cliproxy/service.go:33-38）

### 3.2 `TokenStore + AuthDir`：凭据来源比看起来复杂

从启动逻辑看，项目会先根据环境决定持久化后端，再把统一的 token store 注册成全局实现；如果没有显式外部后端，就退回本地文件存储。（cmd/server/main.go:179-227；cmd/server/main.go:236-391；cmd/server/main.go:447-456）

随后 `Builder.Build()` 会把当前 `cfg.AuthDir` 注入到支持 `SetBaseDir` 的 token store 里，因此“auth 存在哪”不是单靠配置文件决定，而是“配置 + 已选择的存储后端 + Builder 注入”共同决定的。（sdk/cliproxy/builder.go:203-208）

### 3.3 `Builder`：装配边界，而不是语法糖

`Builder` 保存的不是几个可选参数，而是一整套运行时依赖：`cfg`、`configPath`、token/apiKey provider、watcherFactory、hooks、auth/access/core manager、serverOptions。（sdk/cliproxy/builder.go:21-50）

`Build()` 也不是“new 一个对象”这么简单。它会校验 `cfg` 与 `configPath`，补默认 provider/manager，注册 access provider，按 `cfg.Routing.Strategy` 选择 `RoundRobinSelector` 或 `FillFirstSelector`，再把 `RoundTripper`、`Config`、`OAuthModelAlias` 绑定进 `coreManager`，最后组装成 `Service`。（sdk/cliproxy/builder.go:166-241）

### 3.4 `Service`：真正的运行体

`Service` 把进程级运行时统一包起来：当前配置与锁、HTTP server、pprof server、watcher、auth 更新队列、auth/access/core manager、websocket gateway，全都挂在这里。（sdk/cliproxy/service.go:29-92）

这意味着：一旦你已经知道自己处于 server/TUI/cloud/login 的哪条分支，接下来最值得深读的通常不是 `main.go`，而是 `Service.Run()` 和 `Service.Shutdown()`，因为真正的启动顺序、后台 goroutine、热更新与清理逻辑都在这里。（sdk/cliproxy/service.go:466-769）

### 3.5 `coreManager + registry + watcher`：这是一个动态系统

`Service` 里有一条很重要但容易忽略的线：auth 不是静态一次性加载，而是可以被增量更新。`authUpdates` 队列接收 add/modify/delete 事件，`handleAuthUpdate()` 再把它们投射到 `coreManager` 上。（sdk/cliproxy/service.go:72-76；sdk/cliproxy/service.go:114-200）

同一时间，模型目录更新也不是被动等待重启；`registry.SetModelRefreshCallback()` 会在模型定义变化时，重新为受影响 provider 的 auth 注册模型。（sdk/cliproxy/service.go:555-590；internal/registry/model_updater.go:101-138）

把这两条线合起来看，才能理解这个项目为什么不是“读配置 -> 启一次 -> 永远不变”的静态 server，而是带持续更新能力的运行时。（internal/watcher/config_reload.go:79-135；sdk/cliproxy/service.go:611-689）

## 4. 推荐阅读顺序

### 顺序 1：先看产品面，不要一上来钻实现

先读 README 的功能区和 SDK 文档入口，建立边界感：这个项目覆盖哪些 provider、哪些客户端场景、哪些扩展能力，避免把它误读成“只支持 OpenAI-style chat/completions 的小代理”。（README.md:40-89）

### 顺序 2：用 `go.mod` 建立技术地图

第二步看 `go.mod`，目的是快速定位子系统：HTTP、TUI、文件监听、对象存储、数据库、Git、OAuth、websocket。这样你在后面看到目录时，能马上把它们映射回具体能力，而不是把所有目录当成同一层级。（go.mod:0-31）

### 顺序 3：读 `cmd/server/main.go`，搞清楚“有哪些启动形态”

重点不是每一行 flag，而是看它如何：解析模式、探测环境、挑选后端、加载配置、再分派到 login/TUI/server/cloud 等分支。这一步读完，你会知道项目的外部行为面。（cmd/server/main.go:53-98；cmd/server/main.go:161-418；cmd/server/main.go:461-581）

### 顺序 4：立刻跳到 `internal/cmd/run.go`

这是很有价值的一跳，因为它会把你从 CLI 分支逻辑，带到真正的 SDK 运行时。你会看到 `StartService()` / `StartServiceBackground()` 只是薄封装，核心还是 `cliproxy.NewBuilder().Build().Run(...)`。（internal/cmd/run.go:19-84）

### 顺序 5：读 `sdk/cliproxy/builder.go`

这一步是“识别装配边界”。看清楚哪些依赖必须提供，哪些由默认实现兜底，selector 在哪儿选出来，server option 在哪儿注入进来。（sdk/cliproxy/builder.go:18-241）

### 顺序 6：读 `sdk/cliproxy/service.go`

先读结构体与 auth 更新相关段落，再读 `Run()` / `Shutdown()`。这会让你知道 server、watcher、websocket、model refresh、pprof、auto-refresh 是如何一起跑起来的。（sdk/cliproxy/service.go:29-92；sdk/cliproxy/service.go:114-319；sdk/cliproxy/service.go:466-769）

### 顺序 7：最后再深挖配置、存储和 watcher

当你已经知道“系统怎么跑”，再看 `internal/config`、`sdk/auth`、`internal/store`、`internal/watcher`，你会更容易把配置字段、后端持久化和热更新行为放回到正确上下文里。（internal/config/config.go:541-655；sdk/auth/filestore.go:20-170；internal/store/postgresstore.go:27-173；internal/store/objectstore.go:29-155；internal/store/gitstore.go:26-212；internal/watcher/config_reload.go:28-135）

### 如果你的目标不是一样的，阅读顺序也该调整

- **想嵌入 SDK**：从 `internal/cmd/run.go` -> `sdk/cliproxy/builder.go` -> `sdk/cliproxy/service.go` 开始。（internal/cmd/run.go:19-84；sdk/cliproxy/builder.go:18-241；sdk/cliproxy/service.go:466-769）
- **想排查部署/配置问题**：从 `cmd/server/main.go` -> `internal/config/config.go` -> `internal/store/*` 开始。（cmd/server/main.go:161-459；internal/config/config.go:541-655；internal/store/postgresstore.go:145-173；internal/store/objectstore.go:140-155；internal/store/gitstore.go:90-212）
- **想排查热更新/模型变化问题**：从 `sdk/cliproxy/service.go` -> `internal/watcher/*` -> `internal/registry/model_updater.go` 开始。（sdk/cliproxy/service.go:114-319；sdk/cliproxy/service.go:611-689；internal/watcher/watcher.go:30-61；internal/watcher/config_reload.go:28-135；internal/registry/model_updater.go:101-138）

## 常见误解

### 误解 1：这是一个只有 HTTP 层的薄代理

不对。README 已经说明它还包含 OAuth 登录、多账户路由、Amp CLI 管理面、SDK 嵌入能力；`go.mod` 也显示它内建 TUI、文件监听、数据库/Object Store/Git 等支撑能力。（README.md:40-89；go.mod:0-31）

### 误解 2：`sdk/` 只是给外部调用者看的，和 CLI 主程序无关

不对。CLI 自己就是通过 `cliproxy.NewBuilder()` 和 `Service.Run()` 启动的，`internal/cmd/run.go` 只是把 CLI 参数转进 SDK。（internal/cmd/run.go:27-55；sdk/cliproxy/builder.go:166-241）

### 误解 3：多账户只是“多个 API key 轮询”

不对。运行时至少包含 selector、core auth manager、全局模型注册与动态 auth 更新；它能处理的 auth 来源明显不止静态 API key。（README.md:51-60；sdk/cliproxy/builder.go:210-223；sdk/cliproxy/service.go:114-319）

### 误解 4：后台只有一个 API server 在跑

不对。默认 server 形态下，至少还可能有管理面静态资源更新器、模型目录更新器、文件 watcher、core auth 自动刷新，以及按配置启用的 pprof/websocket 相关逻辑。（cmd/server/main.go:499-505；cmd/server/main.go:577-581；internal/managementasset/updater.go:57-111；internal/registry/model_updater.go:73-138；sdk/cliproxy/service.go:605-689）
