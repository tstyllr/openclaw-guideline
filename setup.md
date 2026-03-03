# OpenClaw常用操作

## 新增agent

```bash
openclaw agents add qianqian --workspace ~/.openclaw/workspace-qianqian
```

## 新增telegram channel

```bash
openclaw channels add --channel telegram --token 8722315882:AAEqHLHvCtKdhCeIg9h5U0JPEitybEh9Mzg --account qianqian
```

## Agent 绑定 channel （手动）

在openclaw.json第一层（和agents并列）中添加：

```json
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

授权后，可以在openclaw.json > agents > defaults > models 中添加支持的大模型：

```json
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
