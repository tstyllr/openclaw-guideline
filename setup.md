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
      "groupAllowFrom": ["oc_xxx"],
      "groups": {
        "oc_xxx": {
          // Feishu user IDs (open_id) look like: ou_xxx
          "allowFrom": ["ou_user1", "ou_user2"],
          "requireMention": false
        }
      }
    }
  }
}
```
