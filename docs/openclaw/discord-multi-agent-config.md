# Discord 配置经验总结(OpenClaw 多 Agent)

最后更新：2026-04-01（晚）

## 适用场景

- OpenClaw 多账号(main / architect / pm / dev / qa)
- Discord 群聊协作
- 需要 agent 间互相点名触发

## 一、核心结论(先看这个)

> 团队通讯录见内部共享知识库（不在公开 wiki 中发布）

1. **必须使用真实 mention**:`<@USER_ID>`
   - 纯文本 `@名字` 在很多情况下不会出现在 `mentions[]`,触发失败。
2. **bot 间沟通要显式放行**:每个账号配置 `allowBots: "mentions"`
   - 否则 bot 发给 bot 的消息可能被忽略。
3. **群聊触发建议开启 `requireMention: true`**
   - 能有效防止刷屏和误触发。
4. **Message Content Intent 必须正确配置**
   - 缺失时可能出现 gateway `4014`,导致收消息异常。
5. **streaming 建议统一关闭(`"off"`)**
   - 对协作链路验收帮助不大,容易增加排障噪音。

---

## 二、推荐配置要点(实战版)

### 1) 账号绑定

- 通过 `bindings[]` 显式把 Discord account 绑定到对应 agent。
- 建议每个角色一个独立 bot 账号,职责清晰,审计方便。

### 2) 群策略

- `groupPolicy: "open"`
- `guilds: { "*": { "requireMention": true } }`

这样可以保证:
- 被点名才处理
- 普通闲聊不抢答

### 3) bot-to-bot 放行

建议每个 Discord 账号都配置:

```json
"allowBots": "mentions"
```

否则典型现象:
- 人类 @ bot 可以触发
- bot @ bot 不触发

### 4) streaming

统一设置:

```json
"streaming": "off"
```

注意保持顶层和账号级一致,避免行为不一致。

---

## 三、典型故障与排查路径

### 故障 A:@了没反应

优先检查:
1. 消息里是否是 `<@ID>` 真实 mention(看 `mentions[]` 是否非空)
2. 目标 bot 是否在线且 token 正确
3. 是否被 `requireMention` / 群策略过滤

### 故障 B:bot 间互相 @ 不触发

优先检查:
1. 两边账号是否都配置了 `allowBots: "mentions"`
2. 是否真实 mention 到正确 ID
3. 是否命中了预处理日志 `reason: no-mention`

### 故障 C:连接不稳定 / 收不到内容

优先检查:
1. Discord Developer Portal 是否开启 Message Content Intent
2. 日志是否有 gateway `4014`
3. 网关是否 hot reload 成功

---

## 四、验收方法(建议固定流程)

1. 由 A bot 发送一条同时真实 mention B/C/D 的消息。
2. 要求 B/C/D 分别回固定短码(例如 `B-OK`、`C-OK`、`D-OK`)。
3. 用消息读取接口确认:
   - 回复内容
   - `author.id` 对应正确 bot
   - 时间戳与消息 ID 完整可追踪

验收通过标准:
- 连续 1 次全员回复成功 + 记录消息 ID

---

## 五、团队协作规范(强烈建议)

- 沟通指令必须真实 mention 责任人。
- 需要多人动作时,明确列出人名 + 回执格式。
- 不被 mention 不默认需要回复。
- 验收消息保留 message ID,便于追溯。

---

## 六、一句话复盘

> Discord 多 Agent 协作里，最常见问题不是"模型不聪明"，而是"消息触发条件不成立"。
> 把 **真实 mention + allowBots + requireMention + intent** 四件事配正确，链路就稳。

---

## 七、`ignoreOtherMentions` 陷阱（2026-04-01 新增）

### 现象

- `pegasus`（main bot）在 `#project-pegasus` 频道发了含 `<@Dev_ID>` 的消息。
- Dev 没有任何响应。
- 日志显示：`"reason":"no-mention" discord: skipping guild message`。
- 但消息里确实有真实 mention，`mentions[]` 非空。

### 根因

Dev / QA / Doctor 三个账号均配置了：

```json
"guilds": {
  "*": {
    "requireMention": true,
    "ignoreOtherMentions": true
  }
}
```

`ignoreOtherMentions: true` 的含义是：**只处理"消息内容是直接指令给我本 bot"的 mention，忽略消息里 mention 到我的其他情况**。

实际行为：bot 收到消息后，检查「这条消息是不是主要在叫我处理」，判断为否 → skip。
结果：即使消息里有 `<@DevID>`，只要发送方是另一个 bot 且 `ignoreOtherMentions: true`，Dev 就不会响应。

### 修复

删除所有 agent 账号的 `ignoreOtherMentions` 字段，保留 `requireMention: true` 即可：

```json
// 修复前
"guilds": {
  "*": {
    "requireMention": true,
    "ignoreOtherMentions": true   // ← 删掉这行
  }
}

// 修复后
"guilds": {
  "*": {
    "requireMention": true
  }
}
```

### 影响范围

本次受影响账号：`dev`、`qa`、`doctor`（三个均有此配置）。

### 验证

修复后在 `#project-pegasus` 发 `<@Dev_ID>` 测试消息，Dev 正常回复，日志不再出现 `reason: no-mention`。配置支持热重载，无需重启 gateway。

### 一句话

> `ignoreOtherMentions: true` 会让 bot 在 **被 mention 时仍然不响应**，是 bot-to-bot 协作中最隐蔽的坑之一。除非明确需要只响应人类指令，否则不要设置。

---

## 八、多 Agent 协作的正确 Policy 配置模式（2026-04-01）

### 背景

今天花了整整半天在 `allowFrom` / `groupPolicy` / `dmPolicy` / `guilds.users` 上反复折腾，走了很多弯路。这里把最终正确的配置模式和踩过的所有坑记录下来。

---

### 正确配置结构（经验证可用）

```json
{
  "channels": {
    "discord": {
      "groupPolicy": "allowlist",
      "accounts": {
        "<agent>": {
          "allowBots": "mentions",
          "dmPolicy": "pairing",
          "groupPolicy": "allowlist",
          "guilds": {
            "<guild_id>": {
              "requireMention": true,
              "users": ["*"],
              "channels": {
                "*": { "allow": true }
              }
            }
          }
        }
      }
    }
  }
}
```

### 各字段含义与分工

| 字段 | 作用域 | 说明 |
|------|--------|------|
| `groupPolicy: allowlist` | guild 消息入口 | 只接受白名单 guild 的消息，配合 `guilds` 块使用 |
| `dmPolicy: pairing` | DM 入口 | 私信需要配对，防止陌生人 DM |
| `allowBots: mentions` | bot 消息过滤 | 允许 bot 发来的含 mention 的消息触发响应 |
| `guilds.<id>.users` | guild 内人类用户白名单 | `["*"]` 表示该 guild 内所有用户；也可以写具体 ID |
| `guilds.<id>.channels` | 频道白名单 | `{"*": {"allow": true}}` 表示所有频道都开放 |
| `guilds.<id>.requireMention` | 触发条件 | `true` = 必须被 @ 才响应 |
| `allowFrom` | DM sender 白名单 | **只对 DM 生效**，不影响 guild 消息 |

### 关键结论：`allowFrom` 只管 DM，不管 guild

这是今天最大的误解来源。

`allowFrom` 的作用范围是 **DM（私信）**，不是 guild 频道消息。

- 用 `allowFrom` 控制 guild 消息 → **无效**，guild 消息走 `guilds.users` 白名单
- 用 `allowFrom` 放进 bot ID 来让 bot 消息通过 → **错误方向**，bot 消息通过 `allowBots` 控制

**正确分层：**
```
DM 保护     →  dmPolicy + allowFrom（只放真实用户 ID）
Guild 保护  →  groupPolicy + guilds.<id>.users（人类用户白名单）
Bot 消息    →  allowBots: mentions（独立控制，不走 users）
```

### 踩过的坑

#### 坑 1：`allowFrom` 只放 owner，bot 消息被拦

```json
// ❌ 错误：只放了 owner，Pegasus 发的 @Dev 消息被拦
"allowFrom": ["discord:OWNER_USER_ID"]
```

**现象：** Pegasus @Architect/@Dev/@PM/@QA，日志全部 `reason: no-mention`，无人响应。
**原因：** `allowFrom` 对 guild 消息也生效（作为 sender 过滤的第一道门），Pegasus 的 sender ID 不在白名单里直接被拒。
**教训：** 如果要用 `allowFrom` 限制 DM，不要同时用它控制 guild，或者把所有 bot ID 都加进去——但这样维护成本高，不推荐。最好直接用 `dmPolicy: pairing`，不用 `allowFrom`。

#### 坑 2：`guilds.users` 只放 owner ID，bot 发的消息触发不了

```json
// ❌ 错误
"guilds": {
  "GUILD_ID": {
    "users": ["OWNER_USER_ID"]  // 只有 owner
  }
}
```

**现象：** 同上，bot-to-bot mention 全部失效。
**原因：** `guilds.users` 是人类用户白名单，bot 消息走 `allowBots` 路径，两者独立，但 `users` 不包含 owner 时连你自己发消息也触发不了。
**修复：** `"users": ["*"]` 开放所有用户，安全边界靠 `groupPolicy: allowlist` + `guilds` 绑定到具体 guild 来保证。

#### 坑 3：`allowFrom` 的 ID 格式

```json
// ✅ 正确格式（带 discord: 前缀）
"allowFrom": ["discord:OWNER_USER_ID"]

// ❓ 可能有问题（不带前缀）
"allowFrom": ["OWNER_USER_ID"]
```

日志会显示 `discord users resolved: discord:OWNER_USER_ID→OWNER_USER_ID`，说明 `discord:` 前缀是必要的，gateway 会做解析。

#### 坑 4：`channels.*` 通配符 + `requireMention`

```json
// 只对特定频道要求 mention，其他频道不要求
"channels": {
  "*": { "allow": true, "requireMention": true }  // 全部频道都要求 mention
  // 如果想部分频道不要求，需要逐一列出
}
```

`channels.*` 是通配符，会覆盖所有频道。如果想某些频道不需要 mention（比如 main bot 的 general 频道），需要单独配具体 channel ID。

### 最终有效配置模式总结

```
groupPolicy: allowlist          ← guild 消息入口锁
dmPolicy: pairing               ← DM 入口用配对，不用 allowFrom
allowBots: mentions             ← bot 消息开关
guilds.<id>.users: ["*"]       ← guild 内人类用户全开，安全靠 guild 绑定
guilds.<id>.channels.*: allow  ← 频道全开，需要 mention 触发的加 requireMention
allowFrom: 不用                 ← 除非有严格 DM 白名单需求
```

### 一句话

> `allowFrom` 只管 DM，`guilds.users` 管 guild 人类消息，`allowBots` 管 bot 消息，三条路互相独立，搞混任意一条都会导致消息静默丢失。
