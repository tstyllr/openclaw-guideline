# OpenClaw常用操作

## 新增agent

```bash
openclaw agents add xiao-niu --workspace ~/.openclaw/workspace-xiao-niu
```

## 新增telegram channel

```bash
openclaw channels add --channel telegram --token xxx --account xiao-niu
```

## Agent 绑定 channel （手动）

```json
// ~/.openclaw/openclaw.json
{
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "telegram",
        "accountId": "main"
      }
    },
    {
      "agentId": "xiao-lu",
      "match": {
        "channel": "telegram",
        "accountId": "xiao-lu"
      }
    }
  ]
}
```

## 新增大模型提供商

用 openclaw onboard 授权新的大模型提供商

授权后，添加支持的大模型：

```json
// ~/.openclaw/openclaw.json
{
  "models": {
    "anthropic/claude-opus-4-6": {},
    "anthropic/claude-sonnet-4-6": {},
    "anthropic/claude-haiku-4-5": {},
    "minimax-cn/MiniMax-M2.5": {
      "alias": "Minimax"
    },
    "minimax-cn/MiniMax-M2.5-highspeed": {
      "alias": "Minimax-highspeed"
    },
    "openrouter/auto": {
      "alias": "OpenRouter"
    },
    "openrouter/google/gemini-3.1-pro-preview": {}
  }
}
```

## 智能搜索（本地embedding大模型）

```bash
# 第一步：
openclaw config set agents.defaults.memorySearch.provider "local"
# 第二步：
openclaw memory index --verbose
# 第三步：
openclaw memory search "今天发生了什么" --agent main
```

## 群聊

```json
// ~/.openclaw/openclaw.json
{
  "channels": {
    "feishu": {
      "groupPolicy": "allowlist",
      // Feishu group IDs (chat_id) look like: oc_xxx
      // ["*"] means all
      "groupAllowFrom": ["oc_xxx"],
      "groups": {
        "oc_xxx": {
          // Feishu user IDs (open_id) look like: ou_xxx
          // ["*"] means all
          "allowFrom": ["ou_user1", "ou_user2"],
          "requireMention": false
        }
      }
    }
  }
}
```

## 心跳

最简配置：

```json
// ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m", // 改这里，0m = 禁用
        "target": "last"
      }
    }
  }
}
```

every 支持的格式：

- "15m" → 每 15 分钟
- "1h" → 每 1 小时
- "0m" → 禁用心跳

target 的三类选项：

| 值         | 效果                           |
| ---------- | ------------------------------ |
| "last"     | 发到最近使用的渠道（默认值）   |
| "none"     | 只运行，不发任何消息（纯后台） |
| 指定渠道名 | 发到固定渠道                   |

指定渠道名的可选值：

- "telegram"
- "whatsapp"
- "discord"
- "slack"
- "signal"
- "imessage"
- "googlechat"
- "msteams"

指定渠道时还可以加 to 参数指定具体收件人：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "telegram",
        "to": "123456789" // Telegram chat ID
      }
    }
  }
}
```

指定飞书：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "feishu",
        "accountId": "xiao-niu" // Agent ID
      }
    }
  }
}
```

几个实用配置项：

| 配置项         | 说明                                                 |
| -------------- | ---------------------------------------------------- |
| every          | 心跳间隔                                             |
| activeHours    | 只在某时间段内触发（比如 08:00~24:00，避免半夜打扰） |
| model          | 心跳用更便宜的模型跑，省 token                       |
| target: "none" | 只运行不发消息，纯后台                               |

比如只在白天每小时检查一次：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "1h",
        "target": "last",
        "activeHours": { "start": "08:00", "end": "22:00" }
      }
    }
  }
}
```

## Session

───

配置方式：

```json
// ~/.openclaw/openclaw.json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

───

Session Key 是怎么生成的

OpenClaw 根据 session.dmScope 决定 direct message 映射到哪个 session key：

| dmScope          | DM session key 格式                  |
| ---------------- | ------------------------------------ |
| main（默认）     | agent:<id>:main                      |
| per-channel-peer | agent:<id>:<channel>:direct:<peerId> |
| per-peer         | agent:<id>:dm:<peerId>               |

如果你的 openclaw.json 设置的是 per-channel-peer，飞书来的私信会生成 agent:xiao-niu:feishu:direct:ou_xxx 这种独立 key。

───

那 `agent:<id>:main` 从哪来？

main session 是 OpenClaw 在某些场景下默认保留的 fallback session，比如：

• 早期创建时 dmScope 是 main（后来改成 per-channel-peer，但历史记录还在）
• 通过 sessions_send 主动向 xiao-niu 发消息时，会用到 main
• 其他 agent（比如我）通过 API 发消息给 xiao-niu，默认落到 main

───

关键差异对比

| 特性     | agent:xiao-niu:main            | agent:xiao-niu:feishu:direct:ou_xxx |
| -------- | ------------------------------ | ----------------------------------- |
| 来源     | 程序内部 / 跨 agent 通信       | 真实用户从飞书发来的消息            |
| 回复路由 | 发给 ou_xxx（deliveryContext） | 发给 ou_xxx（deliveryContext）      |
| 上下文   | 独立历史记录                   | 独立历史记录                        |
| 隔离性   | ❌ 如果多人用 main 则共享！    | ✅ 每个用户独立隔离                 |
| 使用场景 | agent-to-agent 通信            | 用户日常对话                        |

───
