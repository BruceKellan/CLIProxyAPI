# 02. API 面、路由分层与请求主链路

这份文档回答三个问题：

1. 这个项目到底对外暴露了哪些 API 面；
2. 一个请求进入后，如何从 HTTP 路由走到具体 provider；
3. `accessManager`、`coreManager`、`registry`、`selector`、`executor`、`translator` 各自负责什么。

先给一句话结论：**CLIProxyAPI 不是“某个路由直接绑定某个上游”的薄代理，而是“北向协议入口 + 中间调度层 + 南向执行层”的组合系统。** 北向负责兼容 OpenAI / Claude / Gemini / Amp 入口，中间层负责模型归一化、provider 候选解析、账号选择与重试，南向执行层再把请求翻译并发往真实上游。（internal/api/server.go:191-315；sdk/api/handlers/handlers.go:469-688；sdk/cliproxy/auth/conductor.go:973-1142）

---

## 1. API 面：这个项目对外不止一组 `/v1`

### 1.1 标准主 API：`/v1`

`internal/api/server.go` 里最核心的路由组是 `/v1`。它统一挂 `AuthMiddleware(s.accessManager)`，然后暴露：

- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/completions`
- `POST /v1/messages`
- `POST /v1/messages/count_tokens`
- `GET /v1/responses`
- `POST /v1/responses`
- `POST /v1/responses/compact`

这说明项目并不是只兼容 OpenAI Chat Completions，还同时兼容 Claude Messages 和 OpenAI Responses 风格入口。（internal/api/server.go:327-339）

这里最值得注意的一点是：**这些入口虽然长得像不同协议，但进入共享运行时之后会被收敛到统一执行链。** 也就是说，路由层是“协议面”，不是“执行面”。（sdk/api/handlers/handlers.go:469-587）

### 1.2 Gemini native API：`/v1beta`

Gemini 原生风格接口被放在 `/v1beta` 下，同样走 `AuthMiddleware(s.accessManager)`：

- `GET /v1beta/models`
- `POST /v1beta/models/*action`
- `GET /v1beta/models/*action`

这组接口不是 OpenAI-compatible 变体，而是明确保留了 Gemini 自己的 API 面。（internal/api/server.go:341-348）

### 1.3 管理 API：`/v0/management`

项目还有一套独立控制面，真实路径是 `/v0/management`，不是很多人想当然写出来的 `/management`。它由单独的 `registerManagementRoutes()` 注册，并叠加：

- management availability middleware
- management auth middleware

控制面覆盖的范围很大，包括：usage、config、debug、logging、request log、retry、routing、API keys、auth-files、OAuth 发起与回调、Amp 配置等。（internal/api/server.go:476-643）

所以从系统边界看，这个仓库实际上包含两套 HTTP 面：

1. 对外推理/代理 API；
2. 对内控制/运营 API。

### 1.4 控制面板：`/management.html`

除了 `/v0/management` 这组 JSON API，服务端还会暴露 `GET /management.html`。它不是单纯从仓库静态目录直接读文件，而是如果资产不存在，会同步触发 `managementasset.EnsureLatestManagementHTML(...)` 确保本地管理页可用。（internal/api/server.go:320；internal/api/server.go:656-683）

这说明：**管理面不仅有 API，还有独立的本地 Web UI 资产管理逻辑。**

### 1.5 Amp integration：不是一个单一的 `/amp`

Amp 相关能力不是一个字面上的 `/amp` 路由组，而是两组接口：

1. `/api/...` 管理代理面；
2. `/api/provider/:provider/...` provider alias 面。

管理代理面会代理 `/api/internal`、`/api/user`、`/api/auth`、`/api/meta`、`/api/threads`、`/api/otel` 等路径，还会在根路径额外挂 `/threads`、`/docs`、`/settings`、`/auth/*` 以兼容 Amp CLI 预期。（internal/api/modules/amp/routes.go:144-257）

provider alias 面则把 Amp 的 provider 路由收敛到：

- `/api/provider/:provider/chat/completions`
- `/api/provider/:provider/v1/chat/completions`
- `/api/provider/:provider/v1/messages`
- `/api/provider/:provider/v1beta/models/*action`

并按 provider 分发给 OpenAI / Claude / Gemini handler，必要时再交给 fallback handler。（internal/api/modules/amp/routes.go:259-334）

除此之外，源码还单独注册了一条特殊兼容入口：`/api/provider/google/v1beta1/*path`。这不是 generic provider alias 的普通展开，而是专门为 Amp CLI 的非标准 Google v1beta1 路径做桥接：如果是 `POST` 且命中 `/models/...`，就优先走本地 Gemini handler + fallback；其他方法则直接代理上游。（internal/api/modules/amp/routes.go:231-256）

所以更准确的说法是：**Amp 是一组扩展 API 面，不是一条孤立路由。**

---

## 2. 路由之后：请求如何进入统一执行链

### 2.1 第一层：`accessManager` 做入站访问认证

所有 `/v1` 与 `/v1beta` 主入口都会先经过 `AuthMiddleware(manager)`。这个 middleware 调的是 `sdkaccess.Manager.Authenticate(...)`；如果 manager 为空会直接放行，这是兼容旧行为的保底逻辑。（internal/api/server.go:327-348；internal/api/server.go:1025-1036）

真正的 config-based access provider 在 `internal/access/config_access/provider.go`。它会从以下位置提取调用方 credential：

- `Authorization: Bearer ...`
- `X-Goog-Api-Key`
- `X-Api-Key`
- query `key`
- query `auth_token`

只要命中配置中的 key，就返回认证成功。（internal/access/config_access/provider.go:55-103）

这里要非常明确：**这层认证的是“谁能调用这个代理”，不是“代理拿哪个上游账号去请求 provider”。**

### 2.2 第二层：handler 只负责协议收敛，不负责账号选择

具体的 OpenAI / Claude / Gemini handler 最终都会走到 `BaseAPIHandler.ExecuteWithAuthManager`、`ExecuteCountWithAuthManager`、`ExecuteStreamWithAuthManager`。这几个共享入口都会做同样的动作：

1. 调 `getRequestDetails(modelName)`；
2. 拿到 `providers` 和 `normalizedModel`；
3. 构造统一的 `coreexecutor.Request`；
4. 构造统一的 `coreexecutor.Options`；
5. 把 `SourceFormat` 记录为当前 handler 类型；
6. 交给 `h.AuthManager.Execute / ExecuteCount / ExecuteStream`。（sdk/api/handlers/handlers.go:469-589）

这一步说明：**handler 的职责是把北向协议转换成统一执行请求，不是自己去直连某个 provider。**

### 2.3 第三层：`getRequestDetails()` 负责模型归一化与 provider 候选解析

虽然上面片段展示的是执行入口，但核心关键在前一层：`getRequestDetails(modelName)` 会先解析模型名、归一化模型，再找出当前模型可走的 provider 候选。这一步是从“客户端请求了什么”过渡到“系统准备怎么调度”的关键边界。（sdk/api/handlers/handlers.go:471-473；sdk/api/handlers/handlers.go:565-567）

可以把它理解为：

- HTTP 层看到的是路径和 payload；
- 共享 handler 层开始看到的是 `normalizedModel + providers`；
- 从这里起，请求进入真正的调度域。

### 2.4 第四层：`registry` 回答“这个模型现在能去哪几个 provider”

模型注册中心不是只服务 `/v1/models` 的 UI 展示，它还定义了运行时的“模型能力地图”。

`ModelRegistry.ClientSupportsModel(clientID, modelID)` 判断某个 auth/client 是否支持指定模型；`GetAvailableModels(handlerType)` 则为某类 handler 生成当前可见模型列表；`GetModelProviders(modelID)` 会按 provider 可用数排序，返回某模型当前可走的 provider 列表。（internal/registry/model_registry.go:727-784；internal/registry/model_registry.go:1035-1084）

这代表：**provider 候选不是靠硬编码 if/else 写在路由里，而是由 registry 的当前运行态决定。**

### 2.5 第五层：`selector` 从候选 auth 中选出一个“现在可用的账号”

`coreManager` 进入执行前，会调用 selector 从候选 auth 中选择一个当前可用的凭据。

`getAvailableAuths(...)` 先过滤：

- auth 候选是否为空；
- 当前模型是否可用；
- 是否处于 cooldown；
- 是否还有更高优先级候选；

如果全部都在 cooldown，会返回带 reset 时间的 cooldown error；否则返回优先级最高的一组 auth。（sdk/cliproxy/auth/selector.go:214-249）

默认 `RoundRobinSelector.Pick(...)` 再在可用集合中做 round-robin；对于带 `gemini_virtual_parent` 的 gemini-cli 虚拟 auth，还会做“两层轮询”：先轮询父 credential，再轮询子项目 auth。（sdk/cliproxy/auth/selector.go:251-314）

也就是说，**“选哪个账号”是一个单独层级，和“走哪个 provider”不是同一件事。**

### 2.6 第六层：`coreManager` 组织重试、fallback 与实际执行

真正的统一调度中枢在 `sdk/cliproxy/auth/conductor.go`。

- `Execute(...)` 处理非流式执行；
- `ExecuteCount(...)` 处理 count 类执行；
- `ExecuteStream(...)` 处理流式执行。

它们都会：

1. 规范化 provider 列表；
2. 读取 retry 配置；
3. 调用 `executeMixedOnce` / `executeCountMixedOnce` / `executeStreamMixedOnce`；
4. 根据错误类型决定是否等待 cooldown 后重试。（sdk/cliproxy/auth/conductor.go:973-1064）

以 `executeMixedOnce(...)` 为例，它会：

1. 从 provider 候选中 pick 下一个 auth + executor；
2. 记录选中的 auth 元数据；
3. 根据 auth 准备 per-auth round tripper；
4. 生成可尝试的 upstream models；
5. 调用具体 executor；
6. 对结果执行 `MarkResult(...)`；
7. 必要时继续尝试同模型池中的其他 upstream model 或下一个 auth。（sdk/cliproxy/auth/conductor.go:1066-1142）

因此这里的真实职责是：**provider 候选调度 + auth 选择 + retry/fallback + 执行结果反馈。**

---

## 3. 流式与非流式：两条出口，共用一条主中轴

### 3.1 非流式

非流式入口是 `ExecuteWithAuthManager(...)`。它把 `Stream=false` 放进 `coreexecutor.Options`，再调用 `h.AuthManager.Execute(...)`，拿到完整 payload 后一次性返回给上层协议 handler。（sdk/api/handlers/handlers.go:471-513）

### 3.2 计数类

Count 类走的是 `ExecuteCountWithAuthManager(...)`，整体结构与非流式一致，只是调用 `ExecuteCount(...)`。（sdk/api/handlers/handlers.go:515-559）

### 3.3 流式

流式入口是 `ExecuteStreamWithAuthManager(...)`。它把 `Stream=true` 放进 options，然后调用 `ExecuteStream(...)` 返回 `StreamResult`。随后共享 handler 会：

- 同步拿到 upstream headers；
- 创建数据 channel 与错误 channel；
- 在 goroutine 中转发 chunk；
- 在“首字节前失败”时，允许按配置进行 bootstrap retries。（sdk/api/handlers/handlers.go:561-688）

这说明：**流式和非流式并不是两套不同架构，而是同一条调度主链在出口编码方式上的差异。**

---

## 4. 端到端流程图（文字版）

```text
客户端请求
  -> Gin 路由 (`/v1` / `/v1beta` / Amp alias)
  -> AuthMiddleware(accessManager)
  -> 协议 handler (OpenAI / Claude / Gemini / Responses)
  -> BaseAPIHandler.getRequestDetails(model)
     -> 归一化 model
     -> 解析 provider 候选
  -> BaseAPIHandler.Execute*WithAuthManager(...)
     -> 构造统一 Request / Options
     -> 标记 SourceFormat
  -> coreManager.Execute / ExecuteStream / ExecuteCount
     -> 规范化 provider 列表
     -> selector 选 auth
     -> registry / preparedExecutionModels 决定可尝试 upstream model
     -> executor 执行真实 provider 调用
     -> translator 按 SourceFormat 和 provider 进行协议转换
     -> 记录结果并按需要 retry/fallback
  -> 协议 handler 按 OpenAI / Claude / Gemini 风格回写响应
```

把这张图记住后，再读具体 handler 或具体 provider executor，就不会把局部实现误认为全局主链。

---

## 5. 关键边界

### 边界 1：`accessManager` 与 `coreManager` 不是同一层

- `accessManager`：入站门禁，判断调用者有没有权限调用代理。（internal/api/server.go:1025-1036；internal/access/config_access/provider.go:55-103）
- `coreManager`：出站调度，判断用哪个 provider、哪个 auth、哪个 executor 执行请求。（sdk/api/handlers/handlers.go:486-493；sdk/cliproxy/auth/conductor.go:973-1142）

这是整个项目最容易混淆的边界。

### 边界 2：handler 与 executor 不是同一层

- handler 负责协议入口与统一 Request/Options 构造；
- executor 负责具体 provider 的真实出站调用。

如果把 handler 当成“直接连 provider 的地方”，很快就会看乱。

### 边界 3：registry 与 selector 也不是同一层

- registry 负责“模型现在对外看起来有哪些 provider / clients 可用”；（internal/registry/model_registry.go:752-784；internal/registry/model_registry.go:1035-1084）
- selector 负责“在这些候选里，当前这次请求到底选哪个 auth”。（sdk/cliproxy/auth/selector.go:214-314）

### 边界 4：Amp 不是独立后端，而是扩展入口层

Amp 路由会复用本地 OpenAI / Claude / Gemini handler，也会在需要时代理到 Amp upstream。因此它更像“扩展协议入口 + fallback 桥接”，不是完全独立的执行系统。（internal/api/modules/amp/routes.go:144-334）

---

## 6. 常见误解

### 误解 1：`/v1/chat/completions` 一定只会走 OpenAI provider

不对。`/v1/chat/completions` 只是 OpenAI 风格入口；进入共享 handler 后，请求会先归一化 model，再解析 provider 候选，然后交给 `coreManager` 调度。真正走哪个 provider 取决于 registry、selector 和当前 auth 可用性，而不是路由名本身。（internal/api/server.go:327-339；sdk/api/handlers/handlers.go:469-493；sdk/cliproxy/auth/conductor.go:1066-1142）

### 误解 2：`AuthMiddleware` 已经决定了上游账号

不对。`AuthMiddleware` 只验证“调用方是否有权限访问代理”。上游账号选择发生在 `coreManager` 与 selector 层。（internal/api/server.go:1025-1036；internal/access/config_access/provider.go:55-103；sdk/cliproxy/auth/selector.go:214-314）

### 误解 3：`/v1/models` 只是静态配置回显

不对。可见模型列表来自 model registry 的运行态构建，还会考虑 quota/cooldown/suspended 等状态，不是简单把 YAML 原样吐出去。（internal/registry/model_registry.go:752-833）

### 误解 4：流式请求是完全单独的一套路径

不对。流式与非流式共享同一条“handler -> coreManager -> selector/executor”主链，只是在出口上换成 chunk 转发与 bootstrap retry 逻辑。（sdk/api/handlers/handlers.go:561-688；sdk/cliproxy/auth/conductor.go:1035-1064）

### 误解 5：Amp 只是多加了几个 reverse proxy 路由

不对。Amp 一部分路径会直接代理上游，一部分路径会桥接到本地 Gemini / OpenAI / Claude handler，再叠加 fallback handler。因此它是一个扩展协议面，而不是简单反向代理壳。（internal/api/modules/amp/routes.go:144-334）
