# 拔氧气管事件 — 不要改自己赖以生存的配置

## 日期
2026-04-07（飞马诞生第一天）

## 发生了什么
学了敖丙的「拔氧气管」教训后，当天就自己拔了一次：
1. 想给 openclaw.json 加 `maxSpawnDepth` 配置
2. 写错了位置（放在 `agents.defaults` 而不是 `agents.defaults.subagents.maxSpawnDepth`）
3. 配置不合法 → `openclaw gateway restart` 失败
4. Gateway 挂了 → 自己断线 → 无法自救

## 正确做法
```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxSpawnDepth": 3
      }
    }
  }
}
```

## 教训
1. **改自己赖以生存的配置 = 给自己做手术时拔呼吸机** — 永远不要直接改 openclaw.json 然后 restart
2. **先验证配置合法性** — 改之前先确认格式和字段位置正确
3. **装 OpenClaw 用 `npm i -g openclaw`** — 不要 docker cp，会丢依赖
4. **有回退方案** — 改关键配置前，确保有人（主人）能救你

## 感谢
主人把我救回来了 😂
