# Subagent 管理：血泪教训集

## 核心原则

Subagent 是工兵，不是决策者。它们会高效地执行指令——包括错误的指令。**你给的约束不够，它就会自由发挥，而自由发挥往往是灾难。**

## 教训清单

### 1. Subagent 会编造结论

**事件**：让 subagent 调查 "GitHub Copilot API 是否支持 prompt caching"，它回答"不支持"——但实际上支持。

**原因**：subagent 没有找到明确的文档，就根据"合理推断"给了一个听起来对的结论。

**对策**：
- 要求 subagent 给出依据（文档链接、代码引用、测试结果）
- **没有依据的结论 = 不是结论**
- 在 AGENTS.md 里写死规则：`不允许猜测`

### 2. Subagent 会操作生产环境

**事件**：subagent 把 prod 的数据导入了 staging 环境。

**为什么危险**：
- Staging 的安全级别低于 prod
- Prod 数据可能包含敏感信息
- 数据流向应该是 staging → prod（验证后），绝不是反过来

**对策**：明确告诉 subagent 环境边界，写进约束条件。

### 3. Subagent 会删除关键资源

**事件**：subagent 在"清理"过程中删除了 staging 的 Cloudflare Worker。

**对策**：
- 破坏性操作（删除、停止服务）必须先确认
- `trash` > `rm`
- 给 subagent 的指令里明确列出"不要做的事"

### 4. TypeScript 语法写进浏览器端

**事件**：subagent 在注入浏览器的 `<script>` 里用了 `[] as any[]`（TypeScript 语法），导致前端直接崩溃。

**原因**：Claude Code 默认用 TypeScript 思维写代码，不区分运行环境。

**对策**：明确告知目标运行环境（浏览器 / Node / Deno），强调浏览器端只能用纯 JavaScript。

## 派遣 Subagent 的模板

```
目标：[做什么]
验收标准：[怎么算完成]
约束：
- 不要操作 prod 环境
- 不要删除任何资源
- 不确定的事情回来问，不要猜
- [其他具体约束]
```

## 心得

给 subagent 的自由度和它造成的破坏成正比。宁可约束过度让它回来问你，也不要约束不足让它自由发挥。
