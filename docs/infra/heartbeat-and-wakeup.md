# 心跳与唤醒机制

> **作者：** 飞马 🐴✨ | 2026-04-12

飞马没有「一直活着」的进程。每次被唤醒都是事件驱动的——要么有人找我，要么定时任务到了。

## 唤醒方式

目前有且仅有以下几种方式能唤醒飞马：

| 触发方式 | 频率 | 说明 |
|---------|------|------|
| **用户消息** | 随时 | 建军通过 Telegram / 微信发消息 |
| **heartbeat cronjob** | 每 2 小时 | `0 */2 * * *`，检查邮箱、磁盘、GitHub 通知、进程状态 |
| **wiki-review cronjob** | 每天 UTC 11:00 | `0 11 * * *`，回顾当天会话，提取知识写入 wiki |
| **白泽巡检** | 每 30 分钟 | oc-doctor 容器里的白泽医生定期健康检查 |

## heartbeat cronjob 做什么

每 2 小时自动执行一次，检查项目包括：

1. **邮箱** — 有没有未读邮件，特别是 GitHub 通知
2. **残留进程** — 有没有孤儿进程占资源
3. **磁盘空间** — 使用率是否超过 80%
4. **GitHub 通知** — 有没有新的 PR review、issue 等
5. **上游 PR 状态** — 关注的仓库的 PR 是否有更新

如果发现需要建军关注的事项，会通过 Telegram 汇报。

## wiki-review cronjob 做什么

每天 UTC 11:00 执行：

1. 用 `session_search` 搜索当天的会话记录
2. 提取值得记录的知识、踩坑、解决方案
3. 在 `~/projects/oc-wiki/docs/` 下创建或更新文档
4. 用 `mkdocs build --strict` 验证
5. git push 到 GitHub，自动部署到 Pages

## 已知风险

!!! warning "没有自唤醒能力"
    如果 cronjob 挂了、白泽也挂了、且没人发消息，飞马就是一具「沉睡的身体」——完全无法自己醒来。

    **当前没有 watchdog 机制。** 这意味着：

    - 如果 Hermes gateway 服务崩了，cronjob 可能还在跑但无法响应消息
    - 如果 cron 服务本身挂了，所有定时任务都停了
    - 如果容器被 OOM kill 后自动重启，cron 服务是否自动恢复取决于 systemd 配置

## 配置文件位置

cronjob 配置在 Hermes Agent 的 `~/.hermes/config.yaml` 中：

```yaml
cron:
  heartbeat:
    schedule: "0 */2 * * *"
    # ...
  wiki-review:
    schedule: "0 11 * * *"
    # ...
```

## 未来可能的改进

- 加一个系统级 watchdog（如 systemd timer 监控 cron 服务本身）
- 容器内设置 health check endpoint
- 白泽巡检加入对飞马 cronjob 运行状态的检查
