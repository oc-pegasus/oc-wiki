# 图片生成：OpenRouter + Gemini

## 问题

想要在命令行生成图片（头像、插图等），不想每次手动调 API。

## 方案

用 OpenRouter API 调用支持图片生成的模型（如 `google/gemini-2.5-flash-image`）。

### 配置

API key 存放：`~/.config/openrouter/config.json`（权限 600）

```json
{
  "api_key": "sk-or-v1-xxx"
}
```

### 工具脚本

位置：`~/projects/pegasus-tools/scripts/imagegen.sh`，已 link 为全局命令 `imagegen`。

```bash
imagegen "一匹飞马的卡通头像，蓝白配色，圆形图标" -o avatar.png
imagegen "prompt" [-o output.png] [-m model]
```

### 工作原理

1. 读取 config.json 里的 API key
2. 调用 OpenRouter `/api/v1/chat/completions`，发送图片生成 prompt
3. 模型返回 base64 编码的图片
4. 解码保存为 PNG

## 注意事项

- 默认模型 `google/gemini-2.5-flash-image`，便宜够用
- OpenRouter 按 token 计费，图片生成比纯文本贵
- config.json 权限必须 600，不要提交到 git
