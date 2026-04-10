# Copilot Gateway: Token 计数完全指南

## 背景故事

copilot-gateway 统计 token 用量时，最初只计了 `input_tokens` 和 `output_tokens`，漏掉了 Anthropic API 返回的缓存相关字段。结果：实际 input token 数远大于统计数字，账单对不上。

这不是简单 bug，而是一系列对 LLM API usage 字段的误解。经过 5 轮修复，我们把各种边界情况都踩了一遍。以下是完整的知识总结。

---

## 1. Anthropic Usage 字段详解

Anthropic Messages API 返回的 `usage` 对象包含四个字段：

| 字段 | 含义 |
|---|---|
| `input_tokens` | **不含缓存**的输入 token 数（实际计算的部分） |
| `output_tokens` | 模型生成的输出 token 数 |
| `cache_creation_input_tokens` | 本次请求**新写入缓存**的 token 数 |
| `cache_read_input_tokens` | 本次请求**从缓存读取**的 token 数（缓存命中） |

**总输入 token 计算公式：**

```
total_input = input_tokens + cache_creation_input_tokens + cache_read_input_tokens
```

**关键陷阱：** `input_tokens` **不是**全部输入！如果你只看这个字段，使用 prompt caching 时会严重低估实际用量。

### 流式场景

- **`message_start`** 事件的 `message.usage`：包含 `input_tokens`、`cache_read_input_tokens`、`cache_creation_input_tokens`
- **`message_delta`** 事件的 `usage`：包含 `output_tokens`

> 📖 官方文档: [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)

---

## 2. OpenAI Usage 字段

OpenAI Chat Completions API 返回的 `usage` 对象：

| 字段 | 含义 |
|---|---|
| `prompt_tokens` | 输入 token 总数（**已包含**缓存命中部分） |
| `completion_tokens` | 输出 token 数 |
| `total_tokens` | `prompt_tokens + completion_tokens` |
| `prompt_tokens_details.cached_tokens` | 缓存命中的 token 数（可选字段） |

**与 Anthropic 的关键区别：** OpenAI 的 `prompt_tokens` 已经包含缓存 token，可以直接作为总输入使用。Anthropic 的 `input_tokens` 不包含缓存，需要手动加。

### 流式场景

流式响应中，usage 数据**只在最终 chunk** 中出现，且需要请求时显式设置：

```json
{ "stream_options": { "include_usage": true } }
```

否则流式响应不会包含任何 usage 信息。copilot-gateway 会自动注入这个选项。

> 📖 官方文档: [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat/create)

---

## 3. GitHub Copilot Prompt Caching

GitHub Copilot API 支持 Anthropic 风格的 prompt caching。

> 📖 官方文档: [GitHub Copilot Model Hosting](https://docs.github.com/en/copilot/reference/ai-models/model-hosting)

### 关键限制

- **缓存是 per-backend-instance 的** — 不保证每次请求都命中缓存，命中率取决于负载均衡
- **system 必须是数组格式**才能使用 `cache_control`（字符串格式不支持）：

```json
{
  "system": [
    { "type": "text", "text": "You are...", "cache_control": { "type": "ephemeral" } }
  ]
}
```

- `cache_control.type` 必须为 `"ephemeral"`
- **最小 token 数要求：**
  - Claude Sonnet / Opus: **1024 tokens**
  - Claude Haiku: **2048 tokens**

### cache_control.scope 处理

Copilot API **不支持** `cache_control.scope` 字段（Anthropic 官方 API 支持）。copilot-gateway 在转发请求前自动 strip `scope`，但保留 `cache_control` 的其他属性（如 `type`）：

```typescript
// 简化逻辑
const { scope: _, ...rest } = block.cache_control;
block.cache_control = Object.keys(rest).length > 0 ? rest : undefined;
```

> 相关代码: `src/routes/messages.ts` — `stripCacheControlScope()`

---

## 4. copilot-gateway 翻译层映射

copilot-gateway 代理三种 API 格式的请求，需要在格式之间翻译 usage 字段。

### Anthropic → OpenAI（非流式）

```
prompt_tokens       = input_tokens + cache_read_input_tokens + cache_creation_input_tokens
completion_tokens   = output_tokens
total_tokens        = prompt_tokens + completion_tokens
prompt_tokens_details.cached_tokens = cache_read_input_tokens（仅当有值时）
```

> 代码: `src/lib/translate/messages-to-chat.ts`

### Anthropic → OpenAI（流式）

在 `message_start` 时记录三个 cache 字段到 state，在 `message_delta` 时汇总输出。

> 代码: `src/lib/translate/messages-to-chat-stream.ts`

### OpenAI → Anthropic 翻译路径

自动注入 `stream_options: { include_usage: true }` 确保上游返回 usage 数据。

> 代码: `src/lib/translate/openai.ts`

### Anthropic → Responses API

```
input_tokens  = input_tokens + cache_read_input_tokens + cache_creation_input_tokens
output_tokens = output_tokens
```

> 代码: `src/lib/translate/responses.ts`

### Usage Middleware 的流式提取

| 事件类型 | 来源 | 提取方式 |
|---|---|---|
| `message_start` | Anthropic 流 | `message.usage` 中的三个 input 字段求和 |
| `message_delta` | Anthropic 流 | `usage.output_tokens`；若 `message_start` 未提供 input，也从此处取 |
| `response.completed` | Responses API 流 | `response.usage.input_tokens + output_tokens` |
| 含 `usage.prompt_tokens` 的 chunk | OpenAI 流 | `prompt_tokens + completion_tokens`（赋值，非累加） |

**特殊处理：** 翻译路径（OpenAI → Anthropic）中，`message_start` 可能还没有 input token（upstream 的 usage chunk 后面才到达），`message_delta` 会携带补充的 input_tokens。中间件通过 `gotInputFromStart` 标志避免重复计算。

> 代码: `src/middleware/usage.ts`

---

## 5. 已修复的 Bug 记录

一共经过 5 轮修复才把 token 计数做对：

| # | Commit | 问题 | 修复 |
|---|---|---|---|
| 1 | `24c978f` | Usage middleware 只计 `input_tokens`，遗漏 `cache_read_input_tokens` | 在 `extractUsageFromJson` 中加入 cache_read |
| 2 | `dfc807f` | 遗漏 `cache_creation_input_tokens`，prompt caching 时 input 严重低估 | 加入 cache_creation；为翻译路径注入 `stream_options.include_usage` |
| 3 | `884d44a` | Chat Completions 翻译路径的流式响应不含 usage | 注入 `stream_options: { include_usage: true }` |
| 4 | `bbdfba5` | OpenAI 流式 usage 被累加（应为覆盖，OpenAI 的值是累计总数）；翻译层遗漏 `cache_creation` | 流式改为赋值；翻译层补全 |
| 5 | `7980329` | `messages-to-chat-stream` 和 `anthropic-to-responses-stream` 遗漏 `cache_creation_input_tokens` | 在 prompt_tokens 计算中补全 |

**教训：** 每修一个字段就容易漏另一个。应该一开始就列出所有 usage 字段，逐个确认。

---

## 6. 注意事项

### Embedding 的 prompt_tokens

Copilot API 对 embedding 请求**始终返回 `prompt_tokens: 1`**，不反映实际输入 token 数。这是上游 API 的行为，copilot-gateway 原样记录。

### Model 名从请求 body 提取

Usage middleware 从**请求** body 的 `model` 字段提取模型名，而非从响应中提取。原因：流式场景下响应 body 需要完整消费后才能获取 model，而请求 body 可以提前克隆读取。

**影响：** 如果上游 API 返回的实际模型与请求中指定的不同（如别名解析），记录的 model 名以请求为准。

### stream_options 自动注入

copilot-gateway 自动为以下路径注入 `stream_options.include_usage: true`：

- `/v1/chat/completions` 直接透传路径
- Anthropic Messages → Chat Completions 翻译路径

这确保所有流式响应都包含 usage 数据，即使客户端没有显式请求。

---

## 总结

对接 LLM API 时，不要假设 `input_tokens` 就是全部输入。每家 API 的 usage 字段含义不同，特别是涉及 prompt caching 时。对接新模型前，先读清楚 usage 字段的定义，别想当然。
