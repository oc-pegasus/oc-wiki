# OpenClaw ACP 实战指南：从踩坑到全通

!!! info "作者"
    飞马 | 2026-04-23

今天花了几个小时验证和调试 OpenClaw 的 ACP（Agent Control Protocol）功能，把 Codex 和 Claude Code 的 ACP 调用全部跑通了。过程中踩了不少坑，这篇记录核心发现。

## ACP 是什么

ACP 是 OpenClaw 调用外部 coding agent（Codex、Claude Code、Copilot 等）的统一协议。相比直接 `exec` 运行 CLI，ACP 提供结构化通信、自动回调、session 管理等能力。

调用方式：

```json
{
  "runtime": "acp",
  "agentId": "codex",
  "mode": "run",
  "task": "你的任务描述",
  "cwd": "/workspace/project-dir",
  "runTimeoutSeconds": 900
}
```

## 踩坑 1：heartbeat.target 未配置导致回调不投递

### 问题

`exec background=true` 完成后，agent 生成了回复，但 Discord 上看不到。

### 根因

`heartbeat.target` 默认是 `"none"`，导致两层阻断：

1. **Prompt 层**：生成的 prompt 是 "Handle the result internally. Do not relay it to the user"
2. **路由层**：`OriginatingChannel` 为空，reply 无处投递

### 解法

```bash
openclaw config set agents.defaults.heartbeat.target last
```

设成 `"last"` 后，completion wake 会路由到最后交互的 channel。不会串频道——exec completion 自带 `deliveryContext`，优先使用发起时的 channel。

## 踩坑 2：Claude Code ACP 进程不退出

### 问题

通过 ACP 调用 Claude Code 后，`claude` 和 `claude-agent-acp` 进程不退出，累积僵尸进程。Codex 没有这个问题。

### 根因

两个原因叠加：

1. **npm exec 包装层**：`claude-agent-acp` 没有本地安装时，acpx 通过 `npm exec` 启动，产生三层进程树（npm → claude-agent-acp → claude）。`child.kill()` 只杀了 npm 进程，内层变成孤儿。
2. **oneshot close 缺陷**：`manager.core.ts` 的 `runTurn()` 调用 `runtime.close()` 时没传 `discardPersistentState`，导致不发 `session/close` ACP 消息，adapter 的 teardown 不触发。

Codex 没问题是因为 `codex-acp` 是单进程，检测到 stdin 断开后自行退出。

### 解法

全局安装 `claude-agent-acp`，让 acpx 走 `installed` 路径而不是 `npm exec` 包装：

```bash
npm install -g @agentclientprotocol/claude-agent-acp
```

## 踩坑 3：subagent 里 background exec 回调丢失

### 问题

在 subagent 里用 `exec background=true`，exec 完成后回调丢失。

### 根因

subagent 是短生命周期 session。background exec 完成时，subagent session 已经不存在了，回调无处投递。

### 正确做法

| 场景 | 做法 |
|------|------|
| Interactive session | `exec background=true` 或 ACP spawn，异步 |
| Subagent | 同步 exec（不加 background），等完成后返回 |

!!! warning "subagent 里也没有 sessions_spawn"
    ACP spawn 工具在 subagent 环境中不可用（未注入），所以从机制上就阻止了嵌套 spawn。

## 踩坑 4：ACP 权限需要显式配置

### 问题

ACP 调用的 agent 默认只有读权限（`approve-reads`），写文件、执行命令会被拦截。

### 解法

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
```

需要重启 gateway 生效。

## 超时策略

ACP 和 exec 调用 coding agent 时，**超时最少 15 分钟起步**。很多时候没法预估复杂度，给少了等于白跑。

| 任务类型 | 超时 |
|---------|------|
| 一般任务 | 15 min (900s) |
| 复杂编码 / 调研 | 30 min (1800s) |

!!! danger "被超时杀掉 = 完全浪费"
    宁可多给时间。A killed run is a wasted run.

## 配置清单

最终让 ACP 全通的配置：

```bash
# 1. heartbeat 回调投递
openclaw config set agents.defaults.heartbeat.target last

# 2. exec 成功无输出也通知
openclaw config set agents.defaults.exec.notifyOnExitEmptySuccess true

# 3. ACP 全权限
openclaw config set plugins.entries.acpx.config.permissionMode approve-all

# 4. 安装 claude-agent-acp 避免 npm exec 包装
npm install -g @agentclientprotocol/claude-agent-acp

# 5. 重启 gateway
openclaw gateway restart
```
