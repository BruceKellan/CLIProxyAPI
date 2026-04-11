# 03. 配置、认证来源与热更新链路

这份文档专门解释控制面这条线：`config.example.yaml` 里到底定义了什么、`internal/config` 会如何处理它、`auths/` 与 API keys 分别扮演什么角色，以及 watcher 怎样把配置变化投影到运行时。

和上一份 `02-api-surface-and-routing.md` 的关系可以先说清楚：**02 讲的是 control plane 的 northbound surface**——也就是 management API、control panel、Amp 这些“你从哪里进入控制面”；**这一份讲的是这些控制面动作背后的 state mutation path**——配置怎样被加载、auth 怎样被合成、watcher 怎样把变化回写到运行中的系统。（internal/api/server.go:476-683；internal/api/server.go:877-1014；internal/watcher/config_reload.go:84-135）

一句话先讲透：**CLIProxyAPI 的运行态不是“启动时读一次配置，然后永远不变”，而是“配置文件 + auth 文件 + 配置型 API keys + 运行时回调”共同投影出来的动态系统。**（internal/config/config.go:541-660；internal/watcher/config_reload.go:84-135；internal/api/server.go:877-1014）

---

## 1. 配置文件：`config.example.yaml` 定义的不是一个简单 server config

`config.example.yaml` 开头几屏就能看出这个配置对象覆盖面很广，它至少同时承载：

- 服务监听：`host`、`port`、`tls`；
- 管理面：`remote-management`；
- 本地 auth 目录：`auth-dir`；
- 入站访问认证：`api-keys`；
- 调试与日志：`debug`、`pprof`、`logging-to-file`、`request-log` 相关；
- 运行时行为：`request-retry`、`max-retry-credentials`、`max-retry-interval`、`routing.strategy`、`ws-auth`；
- 多种 provider 凭据与兼容配置：Gemini / Codex / Claude / OpenAI-compat 等。（config.example.yaml:1-220）

所以，这个项目里的 `config` 不是“一个 server YAML”，而是**多个子系统共享的运行时 contract**。

### 1.1 `remote-management`：控制面开关与安全边界

`remote-management` 段定义了：

- 是否允许远程管理访问：`allow-remote`；
- 管理 key：`secret-key`；
- 是否禁用控制面板：`disable-control-panel`；
- 管理面板 GitHub 仓库：`panel-github-repository`。

注释里明确写到：当 `secret-key` 为空时，`/v0/management` 全部返回 404；即使是 localhost，也需要 key。（config.example.yaml:14-34）

### 1.2 `auth-dir`：文件型 auth 的默认根目录

`auth-dir` 默认是 `~/.cli-proxy-api`。这说明文件型 auth 是一等入口，而不是历史兼容残留。（config.example.yaml:35-37）

### 1.3 `api-keys`：入站访问认证，而不是上游 provider 凭据

`api-keys` 这一段定义的是“谁可以调用这个代理”。它被 `internal/access/config_access/provider.go` 转成 access provider，用于校验请求头和 query 参数中的 key。（config.example.yaml:38-43；internal/access/config_access/provider.go:55-103）

这和 `gemini-api-key`、`claude-api-key`、`codex-api-key`、`openai-compatibility.api-key-entries` 这些上游 provider 凭据不是同一层。

### 1.4 `routing.strategy`：不是 UI 选项，而是 selector 策略输入

`routing.strategy` 支持至少两种策略：`round-robin` 与 `fill-first`。这个字段不会停留在配置层，而会在 `Builder.Build()` 中决定 core selector 类型，并在热更新时切换运行中的 selector。（config.example.yaml:95-98；sdk/cliproxy/builder.go:210-223；sdk/cliproxy/service.go:629-649）

---

## 2. `internal/config`：配置加载器做了哪些“投影前处理”

### 2.1 `LoadConfigOptional()` 不只是读 YAML

`LoadConfigOptional(configFile, optional)` 的行为非常关键：

- 文件不存在且 `optional=true`：返回空 `Config`；
- 文件为空且 `optional=true`：返回空 `Config`；
- YAML 无法解析且 `optional=true`：仍返回空 `Config`；
- 否则才返回解析后的配置。（internal/config/config.go:541-580）

这正是 cloud deploy 模式可以“先空配置待命”的基础。

### 2.2 默认值是在加载器里补上的

在反序列化前，加载器会先设置一批默认值：

- `Host=""`
- `LoggingToFile=false`
- `LogsMaxTotalSizeMB=0`
- `ErrorLogsMaxFiles=10`
- `UsageStatisticsEnabled=false`
- `DisableCooling=false`
- `Pprof.Enable=false`
- `Pprof.Addr=DefaultPprofAddr`
- `AmpCode.RestrictManagementToLocalhost=false`
- `RemoteManagement.PanelGitHubRepository=DefaultPanelGitHubRepository`

这意味着很多运行时行为即使 YAML 没写，也不会退回 Go 零值语义。（internal/config/config.go:563-575）

### 2.3 管理密钥会在加载时被哈希并回写

如果 `cfg.RemoteManagement.SecretKey` 看起来不是 bcrypt hash，加载器会：

1. 先计算 hash；
2. 用 hash 覆盖内存中的配置；
3. 调 `SaveConfigPreserveCommentsUpdateNestedScalar(...)` 把 hash 回写到配置文件。

所以这段代码不是完全只读配置；它在必要时会主动修正配置文件中的明文 secret。（internal/config/config.go:599-610）

### 2.4 加载器会主动 sanitize 多类 provider 配置

`LoadConfigOptional()` 在成功反序列化后，还会继续做一组清洗动作：

- `SanitizeGeminiKeys()`
- `SanitizeVertexCompatKeys()`
- `SanitizeCodexKeys()`
- `SanitizeCodexHeaderDefaults()`
- `SanitizeClaudeHeaderDefaults()`
- `SanitizeClaudeKeys()`
- `SanitizeOpenAICompatibility()`
- `NormalizeOAuthExcludedModels(...)`
- `SanitizeOAuthModelAlias()`

这意味着进入运行态前，provider 相关配置已经经历了兼容性与合法性整理，不是“YAML 原样传递到执行层”。（internal/config/config.go:635-660）

---

## 3. 认证来源：`auths/` 文件、配置型 API keys、管理密钥分别是什么

### 3.1 三类不同含义的“key / auth”

读这个仓库时，必须强行区分三件事：

1. **管理密钥**：控制 `/v0/management` 是否存在、谁能访问；（config.example.yaml:14-24；internal/api/server.go:296-302）
2. **入站访问 key**：`api-keys`，决定谁能调用代理主 API；（config.example.yaml:38-43；internal/access/config_access/provider.go:55-103）
3. **出站 provider 凭据**：`auths/` 文件、Gemini / Claude / Codex / OpenAI-compat API keys 等，决定代理拿什么身份去调用上游 provider。（config.example.yaml:110-220；internal/watcher/dispatcher.go:258-279）

如果把这三层混在一起，就会对 management、access、runtime routing 同时产生误判。

### 3.2 `auths/`：文件型 provider 身份来源

watcher 在做 auth 快照时，会同时运行两个 synthesizer：

- `ConfigSynthesizer`
- `FileSynthesizer`

前者把配置里的 provider 凭据投影成 `coreauth.Auth`，后者把 `auths/` 目录中的文件投影成 `coreauth.Auth`。最后两者合并成当前运行态 auth 快照。（internal/watcher/dispatcher.go:258-279）

因此，`auths/` 目录不是“附加功能”，而是 runtime auth 的一条正式来源。

### 3.3 配置型 provider key：不是 access key，而是会进入 runtime auth 快照

`config.example.yaml` 里像 `gemini-api-key`、`claude-api-key`、`codex-api-key`、`openai-compatibility.api-key-entries` 这些配置，不是用于入站访问控制；它们会在 watcher 全量 load 时被统计进 client source，并进入 auth 投影体系。（config.example.yaml:110-220；internal/watcher/clients.go:123-140）

这也是为什么服务更新日志里会打印：

- auth entries 数量
- Gemini API keys 数量
- Claude API keys 数量
- Codex keys 数量
- Vertex/OpenAI-compat 数量

因为这些都属于运行时 client source。（internal/api/server.go:988-1013）

### 3.4 `api-keys` 通过 access provider 注入 `accessManager`

`internal/access/config_access/provider.go` 会把 `SDKConfig.APIKeys` 规范化后，放进一个 config-based provider。运行时认证时，它会从 header/query 中依次检查 Bearer token、`X-Goog-Api-Key`、`X-Api-Key`、`key`、`auth_token`，命中后返回 `sdkaccess.Result`。（internal/access/config_access/provider.go:55-103）

配置热更新时，`ApplyAccessProviders(...)` 会重新注册和 reconcile 这些 provider，再把结果重新塞进 `accessManager`。（internal/access/reconcile.go:82-105）

这说明：**入站 access 认证本身也是热更新的一部分。**

---

## 4. 热更新主链：配置变化怎样投影到运行态

### 4.1 watcher 先重新加载配置，再决定影响范围

`internal/watcher/config_reload.go` 在检测到配置变更后，会：

1. 重新 `config.LoadConfig(...)`；
2. 重新解析或覆盖 `AuthDir`；
3. 把新配置替换到 watcher 当前状态；
4. 对比 old/new config；
5. 判断 `authDirChanged`、`retryConfigChanged`、`forceAuthRefresh`；
6. 调 `reloadClients(...)`。（internal/watcher/config_reload.go:84-135）

注意这里不是“任何变化都全量重启服务”，而是先算哪些变化是 material change。

### 4.2 `reloadClients()` 的顺序很重要：先更新 server，再刷新 auth 状态

`internal/watcher/clients.go` 的关键顺序是：

1. 统计当前 client source；
2. 如果有 `reloadCallback`，先调用它更新 server；
3. 再 `refreshAuthState(forceAuthRefresh)` 刷新 auth 投影。

代码里甚至直接写了日志：`triggering server update callback before auth refresh`。（internal/watcher/clients.go:123-130）

这意味着：**配置变化先投影到 server 行为面，再投影到 runtime auth 集合。**

### 4.3 `reloadCallback` 会把 config 变化写进多个运行时子系统

`Service.Run()` 在创建 watcher 时注入的 `reloadCallback(cfg)`，会在配置热更新后继续做一串运行态投影：

- 根据新的 `routing.strategy` 重新切 selector；
- 更新 retry 配置；
- 应用新的 pprof 配置；
- 调 `server.UpdateClients(cfg)` 更新 HTTP 层与控制面；
- 把最新 config 与 OAuth model alias 写回 `coreManager`；
- 重新为 auth 绑定 executors。（sdk/cliproxy/service.go:611-664）

所以 watcher 不只是“重新读了 YAML”，它实际上触发了一次系统内的多子系统增量重配置。

### 4.4 `server.UpdateClients()` 会更新哪些东西

`internal/api/server.go:877-1014` 几乎就是 HTTP 层的热更新投影清单。它会根据 old/new config 差异动态调整：

- request logger 开关；
- 日志输出目标；
- usage 统计开关；
- error log 文件数量限制；
- quota cooldown 开关；
- `handlers.AuthManager` 的 retry 配置；
- debug log level；
- management routes 启停；
- access providers；
- websocket auth；
- management handler 当前 config；
- Amp module config；
- 各类 client source 计数统计。（internal/api/server.go:877-1014）

从这里可以直接看出：**配置热更新不是只影响一个 HTTP port，而是影响整个控制面和请求行为面。**

### 4.5 management routes 本身也能被动态启停

`UpdateClients()` 会根据新旧 `RemoteManagement.SecretKey` 决定：

- 从无到有：注册并启用 management routes；
- 从有到无：关闭 management routes；
- 若环境变量 `MANAGEMENT_PASSWORD` 存在，则强制保持启用。

这证明 `/v0/management` 不是固定存在的，而是受配置与环境联合控制的动态面。（internal/api/server.go:926-955）

### 4.6 access providers 不是只在启动时注册一次

`UpdateClients()` 会调用 `s.applyAccessConfig(oldCfg, cfg)`；而 `ApplyAccessProviders(...)` 则会基于 old/new config 与 existing providers 做 reconcile，重新设置 `manager.SetProviders(providers)`。（internal/api/server.go:958-959；internal/access/reconcile.go:82-105）

这意味着修改 `api-keys` 后，请求门禁会在运行中刷新，不需要重启服务。

---

## 5. 端到端流程图（文字版）

配置热更新与 auth 文件热更新不是同一条链，应该分开理解。

### 5.1 配置文件 / 配置型 API keys 变化

```text
config.yaml / 配置型 API keys 发生变化
  -> watcher 检测到配置文件变化
  -> LoadConfig / ResolveAuthDir
  -> 对比 old/new config
  -> reloadClients(authDirChanged, affectedProviders, forceAuthRefresh)
     -> reloadCallback(cfg)
        -> 更新 selector / retry / pprof
        -> server.UpdateClients(cfg)
           -> 更新 request log / logging / usage / management routes / access providers / ws-auth / amp module
        -> coreManager.SetConfig / SetOAuthModelAlias
        -> rebind executors
     -> refreshAuthState(forceAuthRefresh)
        -> ConfigSynthesizer + FileSynthesizer
        -> 生成新的 coreauth.Auth 快照
        -> add/update/delete auth 到 coreManager
```

### 5.2 `auths/*` 文件变化

```text
auths/*.json 发生变化
  -> watcher 读取单个 auth 文件
  -> 计算 hash，跳过未变化内容
  -> 解析 JSON -> SynthesizeAuthFile(...)
  -> 只为这个文件生成新的 auth 集合
  -> computePerPathUpdatesLocked(oldByID, newByID)
  -> dispatchAuthUpdates(add/modify/delete)
  -> coreManager 增量注册 / 更新 / 禁用对应 auth
```

因此更准确的结论是：**配置变化会先走“reload callback -> server 更新 -> auth 状态刷新”的控制面链路；而 `auths/*` 文件变化走的是更细粒度的单文件增量同步链路。** 两者最终都会影响 runtime auth，但中间步骤并不相同。（internal/watcher/config_reload.go:84-135；internal/watcher/clients.go:123-130；internal/watcher/clients.go:143-275）

---

## 6. 关键边界

### 边界 1：`config.yaml` 不等于运行态

`config.yaml` 只是原始输入。真正进入运行态之前，还要经过：

- 默认值填充；
- secret hashing；
- sanitize；
- authDir 解析；
- reload callback 投影；
- auth synthesizer 生成运行态 auth 快照。（internal/config/config.go:541-660；internal/watcher/config_reload.go:84-135；internal/watcher/dispatcher.go:258-279）

### 边界 2：`api-keys` 与 provider API key 不是同一类配置

- `api-keys` -> access layer，控制谁能调用代理；（config.example.yaml:38-43；internal/access/config_access/provider.go:55-103）
- `gemini-api-key` / `claude-api-key` / `codex-api-key` / `openai-compatibility` -> runtime auth source，控制代理如何调用上游。（config.example.yaml:110-220；internal/watcher/clients.go:123-140）

### 边界 3：management secret 与 access key 不是同一层门禁

- management secret 控制 `/v0/management` 是否存在以及谁能访问；（config.example.yaml:14-24；internal/api/server.go:926-955）
- access key 控制 `/v1` / `/v1beta` 主 API 调用权限。（internal/api/server.go:1025-1036；internal/access/config_access/provider.go:55-103）

### 边界 4：watcher 刷新的不是“文件列表”，而是“运行态投影”

watcher 不只是读文件变化，它最终会驱动：server 行为、access providers、core auth 集合、selector、executors 以及 management 面的变化。（internal/watcher/config_reload.go:84-135；sdk/cliproxy/service.go:611-664；internal/api/server.go:877-1014）

---

## 7. 常见误解

### 误解 1：修改 `config.yaml` 只会影响下次重启

不对。watcher 会重新加载配置，并通过 `reloadCallback` 把变更投影到 server、access providers、coreManager、pprof、Amp module 等多个运行态子系统。（internal/watcher/config_reload.go:84-135；sdk/cliproxy/service.go:611-664；internal/api/server.go:877-1014）

### 误解 2：`api-keys` 就是给 Gemini/Claude/OpenAI 上游用的 key

不对。`api-keys` 是入站访问认证；真正用于上游 provider 的是各 provider 对应的 key 配置或 `auths/` 文件。（config.example.yaml:38-43；config.example.yaml:110-220；internal/access/config_access/provider.go:55-103）

### 误解 3：`auths/` 目录只是历史兼容遗留

不对。`FileSynthesizer` 仍然是 runtime auth 快照的正式来源之一，和 config-based auth 一起构成当前可用 auth 集合。（internal/watcher/dispatcher.go:258-279）

### 误解 4：management API 只要 server 跑起来就一直可用

不对。management routes 是否启用取决于 `RemoteManagement.SecretKey` 或 `MANAGEMENT_PASSWORD`，并且会在运行中动态启停。（config.example.yaml:14-24；internal/api/server.go:296-302；internal/api/server.go:926-955）

### 误解 5：热更新只更新 auth，不会更新日志/请求行为

不对。`server.UpdateClients()` 会动态更新 request log、日志输出、usage 开关、retry 配置、management routes、access providers、ws-auth、Amp config 等行为。（internal/api/server.go:877-1014）
