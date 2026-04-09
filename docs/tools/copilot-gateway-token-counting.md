# Copilot Gateway: Token 计数陷阱

## 问题

copilot-gateway 统计 token 用量时，只计了基础的 `input_tokens` 和 `output_tokens`，漏掉了 Anthropic API 返回的缓存相关字段：

- `cache_read_input_tokens` — 从缓存读取的 input tokens
- `cache_creation_input_tokens` — 写入缓存的 input tokens

结果：实际 input token 数远大于统计数字，账单对不上。

## 根因

Anthropic 的 usage 响应里，`input_tokens` **不包含**缓存部分。完整的 input 计算应该是：

```
total_input = input_tokens + cache_read_input_tokens + cache_creation_input_tokens
```

这不是 bug，是 Anthropic API 的设计。但如果你只看 `input_tokens`，会严重低估实际用量。

## 修复

在 token 统计逻辑中，把三个字段都加起来作为 input total。

## 教训

对接 LLM API 时，不要假设 `input_tokens` 就是全部输入。每家 API 的 usage 字段含义不同：

- **Anthropic**: input_tokens + cache_read + cache_creation = 实际输入
- **OpenAI**: prompt_tokens 通常就是全部（但 cached tokens 也要注意）

对接新模型时，先读清楚 usage 字段的定义，别想当然。
