# ai_docs

本目录存放 AI 协议与能力相关的研究与说明文档。

- **[ai_model_protocol_research.md](./ai_model_protocol_research.md)**：OpenAI / Anthropic Claude / Google Gemini 三家调用通信协议研究（请求响应、流式、工具、计费等）。
- **推理/思维链在各家模型中的差异**（见下文）：推理（thinking/reasoning）在 Claude、OpenAI、Gemini 协议中的表示与跨提供商转换要点。

---

# 推理/思维链在各家模型协议中的差异

**版本**：1.0  
**日期**：2026-02-24  
**覆盖平台**：OpenAI（Chat Completions / Responses API）、Anthropic Claude（Messages API）、Google Gemini（含 Gemini CLI）  
**关联文档**：[ai_model_protocol_research.md](./ai_model_protocol_research.md)

---

## 1. 概述

多家 AI 提供商都支持“推理/思维链”（reasoning / extended thinking）：模型在生成最终答案前，会先输出一段内部推理过程。这段内容对复杂任务、多轮对话和跨模型切换时的**上下文连贯性**很重要，但各家的**协议表示方式完全不同**——请求参数、响应结构、流式事件和计费字段均不一致。不做转换直接透传，会导致上下文丢失、误解或计费统计错误。

本文档单独梳理**推理在各家模型协议中的差异**，并给出跨提供商转换时的要点与映射建议。

---

## 2. Anthropic Claude

### 2.1 请求侧：开启扩展思维

Claude 通过顶级字段 `thinking` 控制是否启用扩展思维及预算：

```jsonc
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 8192,
  "messages": [...],
  "thinking": {
    "type": "enabled",        // "enabled" | "disabled" | "adaptive"
    "budget_tokens": 10000     // 推理可用的 token 数，≥1024，且 < max_tokens
  }
}
```

- **enabled**：开启，可用 `budget_tokens` 指定预算。
- **disabled**：不输出思考内容。
- **adaptive**：由模型自动决定用量（无 `budget_tokens`）。

扩展思维占用 `max_tokens` 配额，即推理 + 正文合计不超过 `max_tokens`。

### 2.2 响应侧：thinking 内容块

非流式响应中，推理内容以 **content 数组中的独立块** 出现：

```jsonc
{
  "id": "msg_xxx",
  "role": "assistant",
  "content": [
    {"type": "thinking", "thinking": "Let me analyze step by step...", "signature": "xxx"},
    {"type": "text", "text": "Based on that, the answer is..."}
  ],
  "stop_reason": "end_turn",
  "usage": { "input_tokens": 100, "output_tokens": 500 }
}
```

- **type**：`"thinking"`。
- **thinking**：推理文本。
- **signature**：可选，用于校验或缓存（部分场景会返回 `redacted_thinking`，仅含 `data`，无明文）。

多段推理会对应多个 `type: "thinking"` 块，顺序与生成顺序一致。

### 2.3 流式：thinking 事件序列

Claude 流式用分层事件描述 thinking：

```
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"Let me "}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"think..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}
```

- **content_block_start**：`content_block.type === "thinking"` 表示开始一段推理。
- **content_block_delta**：`delta.type === "thinking_delta"`，`delta.thinking` 为增量文本。
- **content_block_stop**：该段推理结束。

同一轮回复中可多次出现 thinking → text → thinking，需按 `index` 和事件类型维护状态并拼接。

### 2.4 计费

Claude 的扩展思维 token 与普通 output 一起计入 `usage.output_tokens`，**无单独字段**，按输出 token 全价计费。

---

## 3. OpenAI

### 3.1 请求侧：推理强度（推理模型）

OpenAI 的推理模型（如 o1、o3 系列）通过 **reasoning_effort** 控制推理强度，而不是“预算 token”：

```jsonc
{
  "model": "o1",
  "messages": [...],
  "reasoning_effort": "high"   // "low" | "medium" | "high"
}
```

- Chat Completions / 兼容接口常用 `reasoning_effort`。
- Responses API 中可能有 `reasoning` 相关配置（如 `summary`、`encrypted_content` 等），需以官方最新文档为准。

**与 Claude 的差异**：Claude 用 token 预算（数值），OpenAI 用等级（枚举）。跨平台转换时需要 **budget_tokens ↔ reasoning_effort** 的映射（见第 6 节）。

### 3.2 响应侧：reasoning_content

推理模型在 **message** 中可返回 **reasoning_content**，与普通 **content** 并列：

```jsonc
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "The answer is 42.",
      "reasoning_content": "First we need to..."
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 100,
    "completion_tokens_details": {
      "reasoning_tokens": 80
    }
  }
}
```

- **reasoning_content**：可能为字符串或结构化内容（如数组），依模型与 API 版本而定。
- **content**：面向用户的最终回答。

部分实现中 reasoning 会以 content 数组中的 `type: "reasoning"` 等形式出现，需结合具体 API 版本确认。

### 3.3 流式：reasoning_content 增量

流式时推理内容通过 **delta.reasoning_content** 增量返回：

```jsonc
data: {"choices":[{"delta":{"reasoning_content":"First "},"index":0}]}
data: {"choices":[{"delta":{"reasoning_content":"we need to..."},"index":0}]}
data: {"choices":[{"delta":{"content":"The answer is 42."},"index":0}]}
```

- 先出现 reasoning 增量，再出现 content 增量，需按顺序拼接并区分“推理”与“正文”。

### 3.4 计费

- **usage.completion_tokens_details.reasoning_tokens**：专门统计推理 token。
- 计费上推理 token 可能与普通 output 不同（具体以定价页为准）。

---

## 4. Google Gemini

### 4.1 请求侧：thinkingConfig（Gemini API）

Gemini 通过 **generationConfig.thinkingConfig** 控制推理：

```jsonc
{
  "contents": [...],
  "generationConfig": {
    "maxOutputTokens": 8192,
    "thinkingConfig": {
      "thinkingBudget": 1024,      // 推理 token 预算，-1 表示自动
      "includeThoughts": true     // 是否在响应中返回思考内容
    }
  }
}
```

- **thinkingBudget**：推理部分可使用的 token 数；`-1` 表示由模型自动决定。
- **includeThoughts**：为 `true` 时，响应 **parts** 中会包含 `thought` 类型的 part。

### 4.2 响应侧：parts 中的 thought

推理内容在 **candidates[].content.parts[]** 中以带 **thought** 标记的 part 出现：

```jsonc
{
  "candidates": [{
    "content": {
      "role": "model",
      "parts": [
        {"thought": true, "text": "Let me think step by step..."},
        {"text": "The answer is 42."}
      ]
    }
  }],
  "usageMetadata": {
    "promptTokenCount": 10,
    "candidatesTokenCount": 100,
    "thoughtsTokenCount": 80
  }
}
```

- **thought: true** 的 part：表示该段为推理/思考，对应 Claude 的 thinking 块或 OpenAI 的 reasoning_content。
- **text**：普通输出。

### 4.3 流式

流式时每个 chunk 的 **parts** 中同样可能包含 `thought: true` 的片段，需要按顺序拼接并区分 thought 与普通 text。

### 4.4 计费

- **usageMetadata.thoughtsTokenCount**：单独统计思维链 token。
- 部分定价中思维链 token 有折扣（如折半），需以官方计费说明为准。

### 4.5 Gemini CLI 的差异

Gemini CLI 的请求体通常包在 `request` 下，例如：

- **request.generationConfig.thinkingConfig.thinkingBudget**
- **request.generationConfig.thinkingConfig.thinkingLevel**（如 "low" / "medium" / "high"）
- **request.generationConfig.thinkingConfig.includeThoughts**

响应格式与 Gemini API 类似，也有 thought parts 和 thoughtsTokenCount，转换时需按 CLI 的 JSON 结构读写。

---

## 5. 跨提供商时的兼容性问题

### 5.1 为什么“推理”会成为问题

1. **字段与结构完全不同**
   - Claude：`content[].type === "thinking"`，流式用 `content_block_delta.delta.thinking_delta`。
   - OpenAI：`message.reasoning_content` 或 content 中的 reasoning 块，流式用 `delta.reasoning_content`。
   - Gemini：`parts[].thought === true`，usage 用 `thoughtsTokenCount`。
   - 直接透传一家的响应给另一家，对方无法识别，会导致推理部分被忽略或当普通文本处理。

2. **请求参数语义不同**
   - Claude：`thinking.budget_tokens`（数值）。
   - OpenAI：`reasoning_effort`（low/medium/high）。
   - Gemini：`thinkingConfig.thinkingBudget`（数值或 -1）+ `includeThoughts`。
   - 跨平台时必须做参数映射，否则要么不开推理，要么预算/强度不符合预期。

3. **流式事件模型不同**
   - Claude 是 content_block 级（start/delta/stop），OpenAI 是 choice.delta 级，Gemini 是 parts 级；需要各自维护状态机并输出目标格式的事件或 JSON。

4. **上下文序列化与跨模型切换**
   - 若希望“用 Claude 做规划，再切到 GPT 做代码生成”，必须把 Claude 的 thinking 转成目标模型能理解的表示（例如 OpenAI 的 reasoning_content 或系统/用户消息中的结构化摘要），否则规划阶段的推理在切换后丢失，对话不连贯。

### 5.2 安全与策略注意点

- **redacted_thinking**：Claude 可能只返回脱敏或签名，不返回明文；转换时不应把此类内容当作可读推理传给下游。
- **仅信任 assistant 的 thinking**：在请求翻译时，只应将 **assistant** 消息中的 thinking 块映射到目标格式的 reasoning；user/system 中的 thinking 应忽略，避免注入攻击。

---

## 6. 转换与映射建议

### 6.1 请求侧映射（开启推理）

| 源 | 目标 | 建议 |
|----|------|------|
| Claude `thinking.type=enabled` + `budget_tokens` | OpenAI | 将 budget 映射到 `reasoning_effort`（如 0→low，小预算→medium，大预算→high），或使用内部等级（如 minimal/low/medium/high/xhigh）再映射到 API 枚举。 |
| Claude `thinking.type=adaptive` | OpenAI | 映射为 `reasoning_effort: "high"` 或目标端“自动”等价选项。 |
| Claude `thinking.type=disabled` | OpenAI | 不传 reasoning_effort 或传 low/none（若支持）。 |
| OpenAI `reasoning_effort` | Claude | 将等级映射为 `budget_tokens`（如 low→2048, medium→4096, high→8192），并设 `thinking.type: "enabled"`。 |
| Claude / OpenAI | Gemini | 设置 `thinkingConfig.thinkingBudget` 和 `includeThoughts: true`；若有等级则映射到 Gemini CLI 的 `thinkingLevel`。 |

本项目中 **internal/thinking** 提供 `ConvertLevelToBudget` / `ConvertBudgetToLevel`，用于等级与 token 预算的换算，可在此基础上做各 API 的 reasoning_effort / thinkingConfig 填充。

### 6.2 响应侧映射（推理内容）

| 源 | 目标 | 建议 |
|----|------|------|
| Claude `content[].type=thinking` | OpenAI | 提取 `thinking` 文本，写入 `message.reasoning_content`（或 content 中 reasoning 块）；多段用换行拼接。 |
| OpenAI `reasoning_content` | Claude | 在 assistant 的 `content[]` 中插入 `{"type":"thinking","thinking":"..."}` 块；流式则发 content_block_start(thinking) + content_block_delta(thinking_delta) + content_block_stop。 |
| Gemini `parts[].thought===true` | OpenAI | 将对应 `text` 并入 `reasoning_content`。 |
| Gemini thought parts | Claude | 转为 `content[]` 中的 `type: "thinking"` 块及对应流式事件。 |

### 6.3 流式转换要点

- **有状态**：需在会话或请求级维护“当前是否在推理块内”“当前 block index”等，以便正确发出目标端的 start/delta/stop 或 delta 字段。
- **顺序**：保持推理先于正文的顺序；若目标端不支持推理字段，可将推理拼成一段文本放入系统消息或首条用户消息的说明中，避免丢失。
- **结束**：在收到 finish_reason / message_stop / [DONE] 时，若还有未 flush 的推理缓冲，应先输出再结束。

### 6.4 Usage / 计费字段

| 源 | 目标 | 建议 |
|----|------|------|
| OpenAI `reasoning_tokens` | Claude | Claude 无单独推理 token 字段，可合并进 output_tokens 或忽略。 |
| Claude（仅 output_tokens） | OpenAI | 若有 reasoning_content，可估算或从目标端响应取 reasoning_tokens 回填。 |
| Gemini `thoughtsTokenCount` | OpenAI | 映射到 `completion_tokens_details.reasoning_tokens`（若目标响应有）。 |

---

## 7. 与本项目（CLIProxyAPI）的对应关系

本仓库的协议转换与推理处理大致分布如下：

- **internal/translator**  
  - **openai/claude**：Claude 请求 → OpenAI 时，将 `thinking.budget_tokens` 等转为 `reasoning_effort`；OpenAI 响应 → Claude 时，将 `reasoning_content` 转为 `content_block_delta(thinking_delta)` 及 thinking 块。  
  - **claude/openai**：OpenAI 请求 → Claude 时，将 `reasoning_effort` 转为 `thinking.type` + `budget_tokens`；Claude 响应 → OpenAI 时，将 thinking 块转为 `reasoning_content`。  
  - **gemini/claude**、**claude/gemini**、**gemini/openai**、**antigravity/claude** 等：类似地处理 thinking / thought / reasoning_content 的请求与响应映射。
- **internal/thinking**  
  - 提供 `ConvertLevelToBudget`、`ConvertBudgetToLevel` 及等级枚举（如 `LevelXHigh`），供各翻译器统一做“等级 ↔ 预算”换算。  
  - `ApplyThinking` 等会根据模型与提供商，把统一配置写入对应 API 的 reasoning/thinking 参数字段。

实现跨提供商上下文序列化或“先 Claude 规划再切 GPT 执行”时，只要在序列化/反序列化中显式处理“推理”字段（按本节映射），即可保持对话与推理链的连贯性。

---

## 8. 术语与参考

| 中文 | 英文 | 说明 |
|------|------|------|
| 扩展思维 | Extended Thinking | Claude 的推理输出能力 |
| 推理内容 | Reasoning Content | OpenAI 模型中的推理字段/块 |
| 思维链 | Thinking / Thought | 通用说法；Gemini 用 thought part 表示 |
| 推理预算 | Budget (tokens) | Claude/Gemini 以 token 数限制推理长度 |
| 推理强度 | Reasoning Effort | OpenAI 以等级控制推理深度 |

- Claude Messages API：<https://docs.anthropic.com/en/api/messages>  
- OpenAI Chat Completions / Responses：<https://platform.openai.com/docs/api-reference>  
- Gemini generateContent：<https://ai.google.dev/api/generate-content>  
- 本仓库协议总览：[ai_model_protocol_research.md](./ai_model_protocol_research.md)
