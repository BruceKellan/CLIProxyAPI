# 01. 启动模式、配置加载与存储后端

这份文档只讲一条主线：`cmd/server/main.go` 怎样把一个 CLI 进程变成 login/TUI/server/cloud 之一，以及这些模式背后如何选择配置来源、存储后端和 `cliproxy.Service` 运行体。

一句话总结：`main` 不是在“直接启动一个 HTTP server”，而是在做一个 **启动分发器**——先读 flag 和环境，再决定配置/存储来源，然后把控制权交给登录流程、TUI、cloud standby，或 `sdk/cliproxy` 的 `Service`。（cmd/server/main.go:53-98；cmd/server/main.go:161-581；internal/cmd/run.go:19-84）

## 1. main 启动主线：从 flag 到运行体

把 `cmd/server/main.go` 折成 6 步看，最容易建立全局图。

### 1.1 解析 flag，先决定“可能进入哪些模式”

入口先定义了一组模式 flag：`-login`、`-codex-login`、`-codex-device-login`、`-claude-login`、`-qwen-login`、`-iflow-login`、`-iflow-cookie`、`-antigravity-login`、`-kimi-login`、`-vertex-import`、`-tui`、`-standalone`、`-local-model`，以及 `-config`、`-password`、`-oauth-callback-port` 等运行参数。（cmd/server/main.go:59-98）

这一步很重要，因为后面的所有逻辑都不是“server-only”；`main` 从一开始就在准备多条分支，而不是为某个单一模式做配置。（cmd/server/main.go:53-98）

### 1.2 读取工作目录 `.env`，并建立统一的环境变量读取函数

启动后先拿当前工作目录，再尝试加载 `wd/.env`；如果 `.env` 不存在不会报错退出。随后通过 `lookupEnv()` 统一处理大小写环境变量名，这让存储后端探测不依赖单一种大小写约定。（cmd/server/main.go:155-177）

### 1.3 用环境变量探测存储后端，并识别 cloud deploy

接下来 `main` 通过环境变量决定是否启用 Postgres / Git / Object Store：

- `PGSTORE_DSN` 开启 Postgres store；
- `GITSTORE_GIT_URL` 开启 Git store；
- `OBJECTSTORE_ENDPOINT` 开启 Object Store；
- `DEPLOY=cloud` 则标记 cloud deploy 模式。（cmd/server/main.go:179-234）

注意这里的顺序很关键：**先决定后端，再决定配置文件从哪儿读**。也就是说，配置文件路径不是固定的“本地 `config.yaml`”，而可能来自被选中的后端工作区。（cmd/server/main.go:236-391）

### 1.4 按优先级确定配置来源并加载配置

配置来源优先级在代码里是明确写死的：

1. Postgres store；
2. Object Store；
3. Git store；
4. 显式 `-config` 路径；
5. 工作目录下默认 `config.yaml`。（cmd/server/main.go:236-391）

所有分支最后都会落到 `config.LoadConfigOptional(configFilePath, isCloudDeploy)`，也就是“是否允许空配置”由 cloud deploy 模式决定，而不是由具体后端决定。（cmd/server/main.go:264-265；cmd/server/main.go:327-328；cmd/server/main.go:375-382；internal/config/config.go:541-579）

### 1.5 对运行时配置做归一化，再注册全局 token store

配置读完后，启动逻辑会统一做几件事：

- 开关 usage 统计；
- 设置 quota cooldown 行为；
- 配置日志输出；
- 解析并归一化 `cfg.AuthDir`；
- 保存当前配置给管理面模块；
- 按选中的后端注册全局 token store；
- 注册 access provider。（cmd/server/main.go:420-459）

这一步的含义是：**模式分支之前，运行时所需的“配置 + 凭据后端 + access provider”已经被组装好了**。后面的 server/TUI/login 只是用法不同，不会再重新选择存储后端。（cmd/server/main.go:420-459）

### 1.6 进入具体分支：import / login / cloud / TUI / server

最后一段 if/else 链才是“启动成什么”的决定点：Vertex import、各类 login、cloud standby、TUI client、standalone TUI、默认 server 都在这里分流。（cmd/server/main.go:463-581）

## 2. 配置加载：不是简单读一份 YAML

### 2.1 `LoadConfigOptional()` 才是 cloud 模式成立的关键

`LoadConfigOptional(configFile, optional)` 的语义非常具体：当 `optional=true` 时，如果配置文件不存在、路径是目录、文件为空，甚至 YAML 解析失败，它都返回一个空 `Config`，而不是直接报错。（internal/config/config.go:541-579）

这正是 cloud deploy 模式能“无有效配置先待命”的基础；`main` 在后面会再结合文件是否存在、是否是目录、`cfg.Port` 是否为 0 来判断“当前是否有一份可启动服务的配置”。（cmd/server/main.go:400-418）

### 2.2 配置默认值不是在别处补的，而是在加载时直接填进 `Config`

`LoadConfigOptional()` 在反序列化前就设置了一批默认值：`Host` 默认为空字符串（绑定所有接口）、`LoggingToFile=false`、`Pprof.Enable=false`、`Pprof.Addr` 为默认地址、Amp 管理路由 localhost 限制默认关闭、管理面仓库地址有默认值等。（internal/config/config.go:563-575；internal/config/config.go:618-633）

这意味着读源码时不要假设“没写字段就是零值”；很多运行时行为在配置加载阶段已经被合理化了。（internal/config/config.go:563-633）

### 2.3 加载配置时还会做安全与兼容性处理

如果 `remote-management.secret-key` 不是 bcrypt hash，加载器会把它哈希后写回配置文件，避免下次启动重复处理。（internal/config/config.go:599-610）

此外，加载器还会对 Gemini / Vertex-compatible / Codex / Claude / OpenAI-compat 等配置段做 sanitize，而不是把 YAML 生吞进运行时。（internal/config/config.go:635-655）

### 2.4 `cfg.AuthDir` 不是原样照搬，启动后还会再解析一次

`main` 在所有配置加载完成后，会统一调用 `util.ResolveAuthDir(cfg.AuthDir)`；也就是说，即使某个后端先写入了自己的 `AuthDir`，真正运行前仍然会经过一次路径归一化。（cmd/server/main.go:433-438）

## 3. 存储后端：为什么有人说三类，也有人说四类

严格按 `main` 的显式环境开关看，外部存储后端有三类：**Postgres、Object Store、Git**。（cmd/server/main.go:179-227）

但如果按“进程最终可能使用的 token store”来数，是四类，因为没有任何外部后端时会回退到 **本地文件 FileTokenStore**。（cmd/server/main.go:447-456；sdk/auth/filestore.go:20-29）

所以更准确的说法是：**三类外部后端 + 一类默认本地文件后端 = 四类实际运行形态**。

### 3.1 Postgres store

Postgres store 由 `PGSTORE_DSN` 触发，可选 `PGSTORE_SCHEMA` 与 `PGSTORE_LOCAL_PATH`。启动时会推导一个本地 `pgstore` 工作区，创建 `PostgresStore`，再通过 `Bootstrap()` 把数据库里的 config/auth 同步到本地镜像目录，随后用这份本地 `ConfigPath()` 继续加载配置，并把 `cfg.AuthDir` 切到 store 的 auth 目录。（cmd/server/main.go:179-198；cmd/server/main.go:239-268；internal/store/postgresstore.go:27-45；internal/store/postgresstore.go:47-99；internal/store/postgresstore.go:145-173）

关键点不是“配置存在数据库里”，而是 **数据库后端仍然维护一个本地镜像工作区**，以便继续复用文件式流程。（internal/store/postgresstore.go:36-45；internal/store/postgresstore.go:73-97）

### 3.2 Object Store

Object Store 由 `OBJECTSTORE_ENDPOINT`、`OBJECTSTORE_ACCESS_KEY`、`OBJECTSTORE_SECRET_KEY`、`OBJECTSTORE_BUCKET` 等环境变量触发。启动时会校验 endpoint scheme 只能是 `http` 或 `https`，并在本地创建 `objectstore` 工作区，再通过 `Bootstrap()` 从对象存储拉取 config/auth 到本地镜像目录，最后继续从镜像路径加载配置。（cmd/server/main.go:212-227；cmd/server/main.go:269-335；internal/store/objectstore.go:29-50；internal/store/objectstore.go:53-118；internal/store/objectstore.go:140-155）

这里的关键语义也不是“直接远程读文件”，而是 **对象存储后端同样维护本地镜像，以兼容现有文件式 auth/config 流程**。（internal/store/objectstore.go:42-50；internal/store/objectstore.go:87-117）

### 3.3 Git store

Git store 由 `GITSTORE_GIT_URL` 触发，可配用户名、token 与本地路径。启动时会把本地工作区定位到 `gitstore`，把 auth 目录固定到 `repo/auths`，配置目录固定到 `repo/config/config.yaml`；若仓库或配置缺失，会 clone/init 仓库、复制 `config.example.yaml` 到配置路径，并把初始配置提交回 Git。（cmd/server/main.go:199-211；cmd/server/main.go:336-379；internal/store/gitstore.go:26-37；internal/store/gitstore.go:49-88；internal/store/gitstore.go:90-212）

这意味着 Git store 不只是“把 token JSON 推上去”，而是把 **config + auth 一起放进一个受 Git 管理的工作区**。（cmd/server/main.go:352-370；internal/store/gitstore.go:80-88；internal/store/gitstore.go:112-159）

### 3.4 默认本地文件 store

当 Postgres / Object / Git 都未启用时，`main` 会注册 `sdkAuth.NewFileTokenStore()` 作为全局 token store。（cmd/server/main.go:447-456）

这个文件后端本身只知道“把 auth 存在某个 baseDir 下”，真正的 baseDir 会在 `Builder.Build()` 里从 `cfg.AuthDir` 注入进去；因此本地文件后端的路径是 **配置文件 + Builder 装配** 共同决定的，而不是 `main` 里一次性硬编码的。（sdk/auth/filestore.go:20-41；sdk/cliproxy/builder.go:203-208）

## 4. Builder / Service 装配：CLI 最终怎么落到运行时

### 4.1 `internal/cmd/run.go` 证明 CLI 只是薄封装

`StartService()` 做的事情很直接：构造 `cliproxy.NewBuilder()`，注入 `Config`、`ConfigPath` 和本地管理密码，然后建立信号上下文，必要时再额外挂一个 keep-alive endpoint，最后 `Build()` 出 `Service` 并执行 `Run()`。（internal/cmd/run.go:27-55）

如果传入了 `localPassword`，前台模式还会开启一个 10 秒 idle 的 keep-alive 端点，超时后取消运行上下文并触发退出。（internal/cmd/run.go:36-44）

`StartServiceBackground()` 则是同一条装配链，但放进 goroutine 里，并返回 `cancel` + `done` 供外层控制。（internal/cmd/run.go:58-84）

### 4.2 `Builder` 负责补齐默认依赖，不只是转抄参数

`Builder` 保存了 config、configPath、token/apiKey provider、watcherFactory、hooks、auth/access/core manager、serverOptions 等依赖位点。（sdk/cliproxy/builder.go:21-50）

`Build()` 的几个关键动作是：

- 校验 `cfg` 与 `configPath` 必须存在；
- 为 token/apiKey provider、watcherFactory、auth/access/core manager 提供默认实现；
- 给 token store 注入 `cfg.AuthDir`；
- 按 `cfg.Routing.Strategy` 选择 `RoundRobinSelector` 或 `FillFirstSelector`；
- 给 `coreManager` 注入 `RoundTripperProvider`、`Config`、`OAuthModelAlias`；
- 把所有结果收拢到 `Service`。（sdk/cliproxy/builder.go:166-241）

一个容易混淆但必须分清的点是：**token store** 和 **tokenProvider** 不是一回事。前者解决“auth 持久化在哪里”，后者解决“如何把 token-backed client 装进运行时”。两者都在 `Build()` / `Run()` 流程里出现，但职责不同。（sdk/cliproxy/builder.go:28-35；sdk/cliproxy/builder.go:175-188；sdk/cliproxy/builder.go:203-208；sdk/cliproxy/service.go:42-49；sdk/cliproxy/service.go:505-519）

### 4.3 `Service.Run()` 才是实际启动顺序

`Service.Run()` 的启动顺序很清楚：

1. 确保 auth 目录存在；
2. 应用 retry 配置；
3. 让 `coreManager` 从 auth store 载入已有状态；
4. 让 tokenProvider / apiKeyProvider 加载运行时客户端；
5. 创建 `api.Server`；
6. 配置 websocket gateway；
7. 执行启动前 hooks；
8. 注册模型目录刷新回调；
9. 启动 HTTP server；
10. 应用 pprof；
11. 执行启动后 hooks；
12. 创建 watcher 与 auth update queue；
13. 启动 file watcher；
14. 启动 core auth 自动刷新；
15. 阻塞等待 server 退出或上下文取消。（sdk/cliproxy/service.go:475-698）

这就是为什么调试“启动后为什么行为不对”时，常常应该直接看 `sdk/cliproxy/service.go`，而不是反复在 `main.go` 里追分支。（sdk/cliproxy/service.go:475-698）

### 4.4 服务启动完成后，运行中的对象图长什么样

如果把 `01` 和后面的 `02` 接起来读，最关键的一步是先在脑中建立一张“steady-state 对象图”。否则你会知道“怎么启动”，却不知道“启动完之后，哪些常驻对象在互相配合”。

可以先把运行中的系统粗略理解成这样：

```text
main / StartService
  -> cliproxy.Builder
    -> cliproxy.Service
       -> api.Server           # HTTP 入口与 management 面
       -> accessManager        # 入站请求门禁
       -> coreManager          # 出站 auth / selector / executor 调度中枢
       -> wsGateway            # websocket runtime provider 接入点
       -> watcher              # config/auth 文件监听与 reload 入口
       -> model registry       # 模型可见性与 provider 能力地图
```

这张图的重点不是“字段名记忆”，而是分清 3 个层次：

1. **入口层**：`api.Server` 负责北向 HTTP surface；
2. **数据面**：`accessManager` + `coreManager` 负责请求能否进入、进入后如何执行；
3. **控制面**：`watcher`、model registry、wsGateway 负责把配置变化、auth 变化、运行时 provider 变化投影回系统。

从源码上看：

- `Service` 本身持有 `server`、`watcher`、`authUpdates`、`accessManager`、`coreManager`、`wsGateway` 等常驻对象；（sdk/cliproxy/service.go:32-92）
- `Service.Run()` 会把 `coreManager` / `accessManager` 传进 `api.NewServer(...)`，说明 HTTP 层不是独立系统，而是运行时对象图中的一个节点；（sdk/cliproxy/service.go:523-525）
- 同一个 `Service.Run()` 还会注册 model refresh callback、创建 watcher、接入 websocket gateway，说明模型刷新、热更新、runtime provider 接入都不是“外围脚本”，而是 steady-state 的一部分。（sdk/cliproxy/service.go:530-590；sdk/cliproxy/service.go:611-688）

带着这张图去读下一篇 `02-api-surface-and-routing.md`，会更容易理解“请求是怎样穿过这些对象”的，而不是把 02 误读成一组孤立路由说明。

## 5. 模式分支：server / TUI / cloud / login 各自干什么

### 5.1 Login 家族：不是 server 启动的前置步骤，而是独立分支

在分支链里，`-login`、`-antigravity-login`、`-codex-login`、`-codex-device-login`、`-claude-login`、`-qwen-login`、`-iflow-login`、`-iflow-cookie`、`-kimi-login` 都是直接跳到对应 `cmd.Do*` 流程的独立路径；它们不是“先登录再继续启动 server”的固定前置阶段。（cmd/server/main.go:80-90；cmd/server/main.go:466-489）

### 5.2 Cloud deploy：没有有效配置时进入待命，而不是强行起服务

当 `DEPLOY=cloud` 且没有有效配置文件时，`main` 不会尝试启动 API server，而是进入 `cmd.WaitForCloudDeploy()`，仅等待退出信号。（cmd/server/main.go:229-234；cmd/server/main.go:400-418；cmd/server/main.go:490-495；internal/cmd/run.go:86-98）

这一点很重要：cloud 模式不是“忽略配置错误继续跑”，而是“允许先空配置启动进程，但 API server 不启动”。（internal/config/config.go:541-579；internal/cmd/run.go:86-98）

### 5.3 `-tui`：纯客户端模式

如果启用了 `-tui` 但没有 `-standalone`，程序只会跑一个管理 TUI 客户端，假定 proxy server 已经在外部运行；失败时会直接打印 TUI error。（cmd/server/main.go:568-573）

### 5.4 `-tui -standalone`：TUI + 内嵌本地 server

standalone TUI 是另一条完全不同的路径：它会先启动管理面 auto updater、可选的 model updater，安装日志 hook，生成本地管理密码，后台启动内嵌 server，轮询直到 `GetConfig()` 成功，再拉起 TUI；TUI 退出后再 cancel 后台服务。（cmd/server/main.go:499-567）

这条路径本质上是“本地嵌入式部署 + TUI 前端”，不是单纯的 UI 模式。（cmd/server/main.go:501-559；internal/cmd/run.go:58-84）

### 5.5 默认 server 模式

如果不走任何 import/login/TUI/cloud standby 分支，程序会启动管理面 auto updater、按 `-local-model` 决定是否跳过远程模型目录更新，然后调用 `cmd.StartService()` 进入标准 `Builder -> Service` 启动链。（cmd/server/main.go:496-498；cmd/server/main.go:575-581；internal/cmd/run.go:27-55）

### 5.6 `-local-model` 的真实含义

`-local-model` 不是“切换到本地模型服务”，而是 **只使用内嵌模型目录，跳过远程模型目录更新**。代码里它只影响 `registry.StartModelsUpdater()` 是否启动，以及相关日志提示。（cmd/server/main.go:96-97；cmd/server/main.go:496-505；cmd/server/main.go:577-580；internal/registry/model_updater.go:66-85）

## 6. 后台 updater 与 watcher：启动后还有哪些东西在持续工作

### 6.1 管理面静态资源 updater

`managementasset.StartAutoUpdater()` 会启动一个后台 goroutine：

- 配置路径为空则直接跳过；
- 每 3 小时检查一次；
- 如果 control panel 被禁用，或禁用了 auto update，也会跳过；
- 否则尝试同步最新 `management.html` 到本地 static 目录。（internal/managementasset/updater.go:57-111；internal/managementasset/updater.go:132-177；internal/managementasset/updater.go:179-259）

所以它更新的不是模型，也不是 auth，而是 **管理面前端静态资源**。（internal/managementasset/updater.go:27-33；internal/managementasset/updater.go:179-191）

### 6.2 模型目录 updater

`registry.StartModelsUpdater()` 是另一个 singleton 后台 goroutine。它会在启动时先尝试一次刷新，然后每 3 小时从远端 models URL 拉取目录，比较 provider 级差异，并通知回调哪些 provider 发生了变化。（internal/registry/model_updater.go:16-25；internal/registry/model_updater.go:73-138）

而 `Service.Run()` 注册的回调会据此重新为受影响 auth 绑定模型，所以模型目录变化能够在运行中传播到实际可用 auth 上。（sdk/cliproxy/service.go:555-590）

### 6.3 file watcher：配置热更新和 auth 增量更新

`Watcher` 本身持有 `configPath`、`authDir`、当前配置、hash/内容缓存、auth update queue、runtime auth 集合等状态，并在创建时探测当前 token store 是否支持持久化同步与固定 auth 目录。（internal/watcher/watcher.go:20-61；internal/watcher/watcher.go:88-115）

`Service.Run()` 会用 `watcherFactory` 构造 watcher，并为它提供一个 `reloadCallback()`；这个 callback 不只是替换内存里的 config，还会：

- 根据新旧 `routing.strategy` 切换 selector；
- 更新 retry 配置；
- 更新 pprof 配置；
- 让 server 刷新 clients；
- 把新 config 与 OAuth model alias 注入 `coreManager`；
- 重新绑定 executors。（sdk/cliproxy/service.go:611-664）

真正的 config reload 逻辑在 `internal/watcher/config_reload.go`：它会做 150ms debounce、对文件内容做 hash 比较、重新 `LoadConfig()`、重新解析 auth 目录，并只在检测到 material change 时触发 client reload。（internal/watcher/config_reload.go:28-135）

### 6.4 auth update queue：文件变化和运行时 websocket 走的是同一条更新通道

`Service` 里维护了一个带缓冲的 `authUpdates` 队列，并在后台消费 add/modify/delete 事件，把它们映射到 `coreManager` 的注册/更新/禁用操作上。（sdk/cliproxy/service.go:72-76；sdk/cliproxy/service.go:114-200；sdk/cliproxy/service.go:275-339）

更关键的是，websocket 驱动的 runtime provider 也复用了这条通道：`aistudio-*` websocket 连接建立时会创建 runtime-only auth，断开时则删除对应 auth。（sdk/cliproxy/service.go:204-273）

因此 watcher 体系不只是在监听磁盘文件，它实际上是 **文件变更 + 运行时 websocket provider 变更 的统一增量同步通道**。（internal/watcher/watcher.go:140-159；sdk/cliproxy/service.go:153-172；sdk/cliproxy/service.go:204-273）

## 常见误解

### 误解 1：`DEPLOY=cloud` 会在没有配置时照样起 API server

不对。cloud 模式只是允许空配置进入待命；没有有效配置时，代码会进入 `WaitForCloudDeploy()`，明确说明 API server 不启动。（cmd/server/main.go:400-418；cmd/server/main.go:490-495；internal/cmd/run.go:86-98）

### 误解 2：项目只有三类存储后端

不完全对。三类说法只覆盖了显式外部后端（Postgres/Object/Git）；实际运行还包含默认的本地文件后端，所以从 token store 角度看是四类。（cmd/server/main.go:179-227；cmd/server/main.go:447-456；sdk/auth/filestore.go:20-29）

### 误解 3：这些存储后端只存 token，不管配置

不对。Postgres/Object/Git 三类外部后端都同时管理 config 与 auth，并暴露自己的 `ConfigPath()` / `AuthDir()` 供启动流程继续使用。（cmd/server/main.go:255-268；cmd/server/main.go:319-335；cmd/server/main.go:352-379；internal/store/postgresstore.go:145-173；internal/store/objectstore.go:140-155；internal/store/gitstore.go:80-88）

### 误解 4：`-tui` 一定会顺手启动一个本地 server

不对。只有 `-tui -standalone` 才会启动内嵌 server；单独 `-tui` 是纯管理客户端。（cmd/server/main.go:499-567；cmd/server/main.go:568-573）

### 误解 5：watcher 只负责重载 `config.yaml`

不对。它既处理 config 文件变化，也处理 auth 目录变化，还接住 websocket runtime auth 的增量更新。（internal/watcher/watcher.go:30-61；internal/watcher/watcher.go:140-159；sdk/cliproxy/service.go:114-200；sdk/cliproxy/service.go:204-273）
