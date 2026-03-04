🦞 飞书 Channel 对接行动手册（xiaoshu agent）
───

📦 第一步：安装飞书插件

```bash
openclaw plugins install @openclaw/feishu
```

───

🏗️ 第二步：创建飞书应用

1. 打开飞书开放平台：https://open.feishu.cn/app（国际版 Lark 用 https://open.larksuite.com/app）
2. 点击 创建企业自建应用，填写名称、描述、上传图标
3. 进入 凭证与基础信息，复制保存：
   • App ID（格式：cli_xxx）
   • App Secret

───

🔐 第三步：配置权限

进入 权限管理 → 批量导入，粘贴以下 JSON：

```json
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "application:application.app_message_stats.overview:readonly",
      "application:application:self_manage",
      "application:bot.menu:write",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "event:ip_list",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:resource"
    ],
    "user": [
      "aily:file:read",
      "aily:file:write",
      "im:chat.access_event.bot_p2p_chat:read"
    ]
  }
}
```

───

🤖 第四步：启用 Bot 能力

进入 应用能力 → 机器人：

1. 开启机器人能力
2. 设置机器人名称

───

⚙️ 第五步：在 OpenClaw 中配置 Channel

编辑 ~/.openclaw/openclaw.json，添加飞书 channel 配置，并将 xiaoshu agent 绑定：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "groupPolicy": "allowlist",
      "streaming": "off",
      "proxy": "http://127.0.0.1:7890",
      "accounts": {
        "main": {
          "botToken": "xxx"
        }
      }
    },
    "feishu": {
      "enabled": true,
      "dmPolicy": "pairing",
      "accounts": {
        "xiaoshu": {
          "appId": "xxx",
          "appSecret": "xxx",
          "botName": "小书"
        }
      }
    }
  },
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "telegram",
        "accountId": "main"
      }
    },
    {
      "agentId": "xiaoshu",
      "match": {
        "channel": "feishu",
        "accountId": "xiaoshu"
      }
    }
  ]
}
```

───

🚀 第六步：启动 Gateway

```bash
openclaw gateway restart
openclaw gateway status
```

确认 gateway 正在运行后，继续下一步。

───

📡 第七步：配置事件订阅（⚠️ gateway 必须运行中）

回到飞书开放平台 → 事件订阅：

1. 选择 使用长连接接收事件（WebSocket 模式，无需公网 URL）
2. 添加事件：im.message.receive_v1
3. 保存

───

📤 第八步：发布应用

进入 版本管理与发布：

1. 创建一个版本
2. 提交审核并发布
3. 等待管理员审批（企业自建应用通常自动通过）

───

✅ 第九步：测试 & 配对

1. 在飞书中找到你的机器人，发送一条消息
2. 查看日志确认收到消息：openclaw logs --follow
3. 机器人会回复一个配对码，执行：openclaw pairing list feishu
   openclaw pairing approve feishu <CODE>
4. 配对成功后，即可正常对话 🎉

───

🔧 常用排查命令

| 问题              | 命令                         |
| ----------------- | ---------------------------- |
| 机器人没反应      | openclaw logs --follow       |
| 查看 gateway 状态 | openclaw gateway status      |
| 查看待配对请求    | openclaw pairing list feishu |
| 重启 gateway      | openclaw gateway restart     |

───
