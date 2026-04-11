# 05. 用一个真实请求走完整条链路

这一篇不再按模块讲，而是选一个具体请求，从 HTTP 入口一路跟到上游 provider，再跟回客户端响应。目标是把前面几篇里的抽象，落成一条你可以在脑中“重放”的真实路径。

我选的样例是：

> **客户端向 `/v1/chat/completions` 发一个 OpenAI 风格的非流式请求，但系统最终把它路由到 Gemini provider 执行。**

之所以选它，是因为这条链路同时覆盖了：

- 北向 OpenAI API 入口
- handler 层的统一执行入口
- model -> provider 候选解析
- selector 选 auth
- translator 做 OpenAI -> Gemini 协议转换
- Gemini executor 发起真实上游请求
- 再把 Gemini 响应翻回 OpenAI 格式

也就是说，这条样例最能体现“CLIProxyAPI 不是简单转发，而是协议桥 + 调度器 + executor”的本质。

---

## 1. 样例请求长什么样

假设客户端发来一个标准 OpenAI Chat Completions 请求：

```json
POST /v1/chat/completions
Authorization: Bearer YOUR_PROXY_API_KEY
Content-Type: application/json

{
  "model": "gemini-2.5-flash",
  "messages": [
    {"role": "user", "content": "用三句话介绍 CLIProxyAPI 是什么"}
  ],
  "stream": false
}
```

这个请求之所以适合作样例，是因为：

- 路径是 OpenAI 风格；
- payload 也是 OpenAI 风格；
- 但 `model` 指向的是 Gemini 系模型；
- 所以系统会同时走“OpenAI handler”和“Gemini executor / translator”。

换句话说，这个样例天然展示了“北向协议”和“南向 provider”不是同一层。

---

## 2. 第 0 步：为什么这个请求能被系统接住

在路由注册阶段，`internal/api/server.go` 把 `/v1/chat/completions` 绑定到 `openaiHandlers.ChatCompletions`，并且整个 `/v1` 路由组统一挂了 `AuthMiddleware(s.accessManager)`。（internal/api/server.go:327-339）

所以请求真正进入业务处理前，先要过两关：

1. 路径命中 `/v1/chat/completions`
2. 通过 `accessManager` 的入站访问认证

这两步都在 HTTP 入口层完成，还没有进入 provider 调度。

---

## 3. 第 1 步：`AuthMiddleware` 先判断“你能不能调用这个代理”

`AuthMiddleware(manager)` 会调用 `manager.Authenticate(c.Request.Context(), c.Request)`；如果当前没有任何 provider，则走 legacy 兼容行为直接放行，否则会要求请求携带合法凭据。（internal/api/server.go:1025-1036）

如果你的 key 来自 `config.example.yaml` 里的 `api-keys`，那实际做校验的是 `internal/access/config_access/provider.go`。它会从这些位置提取 credential：

- `Authorization: Bearer ...`
- `X-Goog-Api-Key`
- `X-Api-Key`
- query `key`
- query `auth_token`

只要命中配置中的 key，就返回认证成功。（internal/access/config_access/provider.go:55-103）

这里一定要记住：

> **这一步认证的是“客户端是否能调用 CLIProxyAPI”，不是“CLIProxyAPI 应该拿哪个上游账号去调模型”。**

后者要到更后面的 `coreManager` 才会发生。

---

## 4. 第 2 步：OpenAI handler 接管请求

路由命中后，会进入 `OpenAIAPIHandler.ChatCompletions(c)`。（sdk/api/handlers/openai/openai_handlers.go:92-129）

它先做三件事：

1. `c.GetRawData()` 读取原始 JSON；
2. 看 `stream` 字段，判断流式还是非流式；
3. 如果发现有人把 OpenAI Responses-format payload 发到了 `/v1/chat/completions`，会先做一次 request shape 转换，再继续处理。（sdk/api/handlers/openai/openai_handlers.go:98-127）

对于我们这个样例：

- `stream = false`
- payload 本身就是 chat completions 形态

所以会走 `handleNonStreamingResponse(c, rawJSON)`。（sdk/api/handlers/openai/openai_handlers.go:123-127；sdk/api/handlers/openai/openai_handlers.go:429-443）

这一步还停留在“北向协议入口层”。

---

## 5. 第 3 步：handler 把 HTTP 请求收敛成统一执行请求

`handleNonStreamingResponse` 会：

1. 从 JSON 里取 `model`；
2. 建立带取消能力的执行上下文；
3. 调 `ExecuteWithAuthManager(...)`；
4. 把最终 payload 写回客户端。（sdk/api/handlers/openai/openai_handlers.go:429-443）

真正的统一收敛发生在 `BaseAPIHandler.ExecuteWithAuthManager(...)`：

- 调 `getRequestDetails(modelName)`
- 得到 `providers` 与 `normalizedModel`
- 构造 `coreexecutor.Request{Model, Payload}`
- 构造 `coreexecutor.Options{Stream, Alt, OriginalRequest, SourceFormat}`
- 然后交给 `h.AuthManager.Execute(...)`。（sdk/api/handlers/handlers.go:469-513）

这里有两个关键字段值得记住：

### 5.1 `normalizedModel`
这代表系统内部最终用于调度和执行的模型名，不一定和客户端原始传入值完全一样。

### 5.2 `SourceFormat`
这里会被设置为 `sdktranslator.FromString(handlerType)`。对于 `/v1/chat/completions`，它表达的是：

> 这个请求来自 OpenAI 风格入口。

这会直接影响后面 translator 选哪套变换规则。（sdk/api/handlers/handlers.go:486-491）

所以从这一步开始，请求已经不再只是一个 Gin 请求，而变成了统一运行时里的执行请求。

---

## 6. 第 4 步：系统决定“这个模型可以走哪些 provider”

`getRequestDetails(modelName)` 的核心意义，不是简单返回模型名，而是把“客户端请求了什么”转换成“运行时准备怎么调度”。

虽然我们这里只读到了调用点，但从整个系统结构可以确定它至少完成两件事：

1. 归一化模型名；
2. 找到该模型可走的 provider 候选。

provider 候选能力最终来自 model registry。

`ModelRegistry.GetModelProviders(modelID)` 会按 provider 当前可用数排序，返回某个模型当前可走的 provider 列表；如果这个模型当前在 registry 里没有 provider，返回的就是空。（internal/registry/model_registry.go:1035-1084）

对我们这个样例，直观理解就是：

- 客户端走的是 OpenAI API 面；
- 但因为 `model = gemini-2.5-flash`；
- registry 很可能会把 provider 候选解析成 `gemini`；
- 所以后面真正执行的不是 OpenAI executor，而是 Gemini executor。

这就是“入口协议 ≠ 上游 provider”的最直观例子。

---

## 7. 第 5 步：`coreManager` 开始调度

`ExecuteWithAuthManager(...)` 收敛完以后，会调用 `h.AuthManager.Execute(ctx, providers, req, opts)`。（sdk/api/handlers/handlers.go:493）

这里的 `AuthManager` 实际上是 `coreauth.Manager`，也就是整个出站调度中枢。

`Manager.Execute(...)` 会：

1. 规范化 provider 列表；
2. 读取 retry 设置；
3. 进入 `executeMixedOnce(...)`；
4. 如果发生合适类型的错误，就按 cooldown / retry 规则再试。（sdk/cliproxy/auth/conductor.go:973-1002）

它不是“选一个 provider 直接打出去”，而是一套可重试、可 fallback 的调度器。

---

## 8. 第 6 步：调度器选中“这次应该用哪个 auth”

在 `executeMixedOnce(...)` 里，真正的第一步是：

- `pickNextMixed(ctx, providers, routeModel, opts, tried)`

它会返回：

- 选中的 `auth`
- 对应的 `executor`
- provider 名

然后才开始真正执行。（sdk/cliproxy/auth/conductor.go:1082-1088）

这里有一个容易被旧心智模型误导的点：**当前主路径优先走 scheduler fast path，而不是直接落到 legacy selector。**

- `useSchedulerFastPath()` 会在存在 scheduler 且 selector 仍是 built-in selector 时返回 true；（sdk/cliproxy/auth/conductor.go:2225-2230）
- 此时 `pickNextMixed()` 会直接调用 `m.scheduler.pickMixed(...)`；（sdk/cliproxy/auth/conductor.go:2421-2470）
- 只有不满足 fast path 条件时，才回退到 legacy 的 selector 选择逻辑。（sdk/cliproxy/auth/conductor.go:2421-2424）

也就是说，概念上你仍然可以把这一步理解成“在候选 auth 里选出这次应该用谁”，但实现上当前主路径已经是：

> provider 候选 -> scheduler -> 选中 auth -> 找到对应 executor

`sdk/cliproxy/auth/selector.go` 里的 `getAvailableAuths(...)` 和 `RoundRobinSelector.Pick(...)` 仍然很重要，因为它们代表了系统的选择语义：

- 过滤不可用 / cooldown / 不支持该模型的 auth；
- 在可用集合里按策略选择；
- 必要时按 round-robin 轮换。（sdk/cliproxy/auth/selector.go:214-314）

如果你把这一步换成人话，就是：

> “Gemini 这个 provider 当前有哪些账号能打这个模型？它们谁没在 cooldown？谁优先级更高？这次系统最终挑中了谁？”

所以调度维度已经从“路径”变成了“provider + model + auth 可用性”。

---

## 9. 第 7 步：`coreManager` 为选中的 auth 准备实际执行参数

`executeMixedOnce(...)` 在拿到 `auth` 和 `executor` 之后，还会继续做几件非常关键的事：

1. 把选中的 auth 记到 metadata；
2. 如果这个 auth 有专属 RoundTripper，就把它塞进执行上下文；
3. 调 `preparedExecutionModels(auth, routeModel)`，得到这次真正尝试的 upstream model 列表；
4. 对每个候选 upstream model 调 executor。（sdk/cliproxy/auth/conductor.go:1090-1112）

这一步说明两件事：

### 9.1 不是所有 auth 都直接用同一个底层网络配置
`roundTripperFor(auth)` 可以给不同 auth 提供不同 transport，比如不同代理设置。（sdk/cliproxy/auth/conductor.go:1095-1099）

### 9.2 route model 和 upstream model 可能不是同一个东西
`preparedExecutionModels(...)` 暗示系统支持：

- alias pool
- fallback model
- route model -> upstream model 的映射

也就是说，客户端看到的 model，和最终真正打到 provider 的 model，可能经过了一次或多次内部映射。（sdk/cliproxy/auth/conductor.go:1101-1111）

---

## 10. 第 8 步：Gemini executor 把 OpenAI 请求翻译后发给真正的上游

现在终于进入我们这个样例最关键的一步：

> 北向是 OpenAI handler，南向是 Gemini executor。

Gemini executor 的 `Execute(...)` 做的关键动作非常清晰：

1. 读取 `opts.SourceFormat`，知道原始请求来自什么协议；
2. 定义目标格式为 `sdktranslator.FromString("gemini")`；
3. 先做 request translation：`TranslateRequest(from, to, ...)`；
4. 应用 thinking、payload config、model 字段修正；
5. 组装 Gemini API URL；
6. 注入 API key 或 Bearer token；
7. 发出 HTTP 请求；
8. 收到上游响应后，再做 response translation：`TranslateNonStream(to, from, ...)`；
9. 返回统一响应给上层。（internal/runtime/executor/gemini_executor.go:105-209）

这就是整套系统最核心的“桥接瞬间”。

对我们的样例来说，关键变量会是：

- `from = openai`
- `to = gemini`

因此，executor 会把原始 OpenAI Chat Completions payload 转成 Gemini 的 `generateContent` 请求体，然后把 Gemini 返回再翻译回 OpenAI 风格响应。

---

## 11. 第 9 步：translator 实际做了什么转换

translator 体系的默认注册表在 `sdk/translator/registry.go`，它维护：

- request transforms
- response transforms

如果没有找到转换器，请求会原样返回，但会尽量把 `model` 字段修正成内部解析后的模型值。（sdk/translator/registry.go:27-66）

真正把各种 provider 方向注册进来的，是 `internal/translator/init.go`。其中明确包含：

- `internal/translator/openai/gemini`
- `internal/translator/gemini/openai/chat-completions`
- `internal/translator/gemini/openai/responses`

也就是说，OpenAI <-> Gemini 这条双向转换链在系统启动时就已经注册好了。（internal/translator/init.go:20-30）

对我们这个样例，真正会命中的 request transform 是：

- OpenAI 风格请求进入 Gemini executor
- executor 调 `TranslateRequest(from=openai, to=gemini, ...)`
- translator registry 选中已注册的 OpenAI -> Gemini request transform（sdk/translator/registry.go:27-66；internal/runtime/executor/gemini_executor.go:117-126）

而我们手头直接读到的 `internal/translator/openai/gemini/openai_gemini_request.go` 更适合作为一个“复杂度证明”的反向示例：它展示了 translator 不只是改字段名，而是真的会处理：

- `generationConfig`
- `systemInstruction`
- `contents`
- inline image
- function call / tool call
- function response
- thinkingConfig -> reasoning_effort

（internal/translator/openai/gemini/openai_gemini_request.go:19-260）

所以这里最重要的认知不是某一个函数名，而是：

> **translator 不是简单改个 `model` 字段，而是真正做协议语义映射。**

---

## 12. 第 10 步：Gemini executor 发出真实 HTTP 请求

当 request translation 完成后，Gemini executor 默认会把请求发到官方 Gemini endpoint：

- `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent`

或在流式时发到：

- `...:streamGenerateContent`

但这不是绝对固定值。`resolveGeminiBaseURL(auth)` 会优先读取 auth 里的 `base_url` 覆盖项；只有没有覆盖时，才回退到默认官方地址。（internal/runtime/executor/gemini_executor.go:137-145；internal/runtime/executor/gemini_executor.go:244-245；internal/runtime/executor/gemini_executor.go:449-460）

它会根据 auth 注入：

- `x-goog-api-key`
- 或 `Authorization: Bearer ...`

还会应用额外的 Gemini headers，并通过 `newProxyAwareHTTPClient(...)` 发起真正的 HTTP 调用。（internal/runtime/executor/gemini_executor.go:155-181；internal/runtime/executor/gemini_executor.go:258-260）

到这里为止，CLIProxyAPI 才真正“出网”。

所以如果你要定位问题发生在哪一层，可以这样分：

- 在这一步之前，多半是本地调度 / 映射问题；
- 在这一步之后，多半是上游 provider / credential / 网络问题。

---

## 13. 第 11 步：上游响应再被翻回 OpenAI 风格

Gemini executor 收到 2xx 响应后，会：

1. 读完整 body；
2. 记录 usage；
3. 调 `sdktranslator.TranslateNonStream(ctx, to, from, req.Model, opts.OriginalRequest, body, data, &param)`；
4. 把输出 payload 返回给上层。（internal/runtime/executor/gemini_executor.go:191-209）

这里的参数顺序非常重要：

- request translation 时是 `from -> to`
- response translation 时是 `to -> from`

对我们这个样例而言：

- request: OpenAI -> Gemini
- response: Gemini -> OpenAI

所以最终客户端看到的仍然是 OpenAI Chat Completions 风格 JSON，而不是 Gemini 原始格式。

这也是为什么客户端可以“说 OpenAI”，但后面实际上用的是 Gemini 账号。

---

## 14. 第 12 步：handler 把最终结果写回客户端

等 `ExecuteWithAuthManager(...)` 返回后，`handleNonStreamingResponse(...)` 会：

1. 把 upstream headers 透传到下游（若允许）；
2. 把最终 payload 写进 `c.Writer`；
3. 结束请求上下文。（sdk/api/handlers/openai/openai_handlers.go:433-442）

所以从客户端视角看，整个过程只是：

- 发一个 OpenAI 请求
- 收一个 OpenAI 响应

中间 OpenAI -> Gemini -> OpenAI 的转换、selector 选 auth、executor 出网、retry/fallback，这些细节都被代理层吸收了。

---

## 15. 把这条链压缩成一句话

对于这个样例：

> `/v1/chat/completions` 只是北向入口；真正的执行链是：
> `OpenAI handler -> getRequestDetails -> coreManager -> selector -> Gemini executor -> OpenAI<->Gemini translator -> Gemini API -> translator 回转 -> OpenAI response`。

这句话几乎可以当你之后阅读请求链路源码时的导航句。

---

## 16. 端到端流程图（文字版）

```text
客户端
  -> POST /v1/chat/completions
  -> AuthMiddleware(accessManager)
  -> OpenAIAPIHandler.ChatCompletions
  -> handleNonStreamingResponse
  -> BaseAPIHandler.ExecuteWithAuthManager
     -> getRequestDetails(model)
        -> normalizedModel + providers
     -> coreexecutor.Request / Options
     -> coreManager.Execute
        -> selector 选中某个 gemini auth
        -> preparedExecutionModels
        -> GeminiExecutor.Execute
           -> TranslateRequest(openai -> gemini)
           -> HTTP POST generativelanguage.googleapis.com
           -> TranslateNonStream(gemini -> openai)
     -> 返回统一 payload
  -> OpenAI handler 写回 JSON 响应
  -> 客户端收到 OpenAI 风格结果
```

---

## 17. 读这篇时最该建立的 5 个认知

### 认知 1：路径不决定上游 provider
`/v1/chat/completions` 是 OpenAI 风格入口，不等于一定走 OpenAI provider。

### 认知 2：`accessManager` 不是 provider 调度器
它解决“谁能进代理”，不是“出代理后去哪”。

### 认知 3：`getRequestDetails` 是协议入口和运行时调度之间的分水岭
过了它，请求就进入内部统一执行域。

### 认知 4：`coreManager` 真正负责 auth 选择、retry、fallback
它不是简单的调用封装，而是调度中枢。

### 认知 5：translator 是协议语义映射，不是字符串替换
它负责把 OpenAI / Gemini / Claude / Codex 的结构差异真正桥接起来。

---

## 18. 常见误解

### 误解 1：只要走 `/v1/chat/completions`，后面就是 OpenAI 全家桶
不对。路径只决定北向协议入口，不决定最终 provider。真正的 provider 取决于 model 解析、registry、selector 和 auth 可用性。（sdk/api/handlers/handlers.go:469-513；internal/registry/model_registry.go:1035-1084；sdk/cliproxy/auth/selector.go:214-314）

### 误解 2：translator 只是让字段名兼容
不对。它会处理 tool calls、多模态、thinking、系统消息、stream/non-stream 等协议语义，不是只改几个字段。（internal/translator/openai/gemini/openai_gemini_request.go:19-260；sdk/translator/registry.go:27-66）

### 误解 3：selector 选到 auth 后就不会再 fallback
不对。`coreManager` 在执行阶段仍可能根据错误、cooldown、模型池继续尝试其他 auth 或其他 upstream model。（sdk/cliproxy/auth/conductor.go:1066-1142）

### 误解 4：客户端看到 OpenAI 响应，说明中间没有发生协议转换
不对。恰恰相反，客户端之所以能继续看到 OpenAI 风格结果，正是因为 request 和 response 都经过了 translator。（internal/runtime/executor/gemini_executor.go:117-125；internal/runtime/executor/gemini_executor.go:205-208）

---

## 19. 这篇和前面几篇怎么配合读

- 如果你刚读完 `02-api-surface-and-routing.md`：这一篇相当于把 02 的抽象主链落成一个具体案例。
- 如果你读完后还想更深入：下一步最适合自己去追的源码是：
  1. `sdk/api/handlers/openai/openai_handlers.go`
  2. `sdk/api/handlers/handlers.go`
  3. `sdk/cliproxy/auth/conductor.go`
  4. `internal/runtime/executor/gemini_executor.go`
  5. `internal/translator/openai/gemini/*`

如果你能把这 5 个位置按这篇文档重新走一遍，这个项目最核心的请求路径你就已经真正吃透了。
