# Agent to Agent 通信指南

> 让多个 OpenClaw agent 协同工作，跨渠道、跨职责完成任务。

---

## 概述

OpenClaw 支持多个 agent 并行运行，每个 agent 拥有独立的工作区、权限和渠道账号。**Agent to Agent 通信**（简称 A2A）允许 agent 之间互相发送消息、传递任务结果，实现协作。

核心工具：`sessions_send`

```
sessions_send(
  sessionKey: "agent:<agentId>:main",
  message: "任务内容或问题"
)
```

---

## 应用场景

### 场景一：跨渠道信息搬运

**问题：** 用户在 Telegram 想查飞书上的文档，但主 agent 没有飞书权限。

**解法：**

1. 主 agent（Telegram）收到请求
2. 通过 A2A 发消息给飞书 agent
3. 飞书 agent 查询文档，把结果推回主 agent
4. 主 agent 汇总回复给用户

**价值：** 每个 agent 专注自己的渠道权限，不需要一个 agent 持有所有凭证。

---

### 场景二：任务流水线（职责分离）

**问题：** 用户说"把这段会议记录整理成飞书文档"，需要理解 + 写入两个步骤。

**解法：**

1. 主 agent 负责理解意图、整理内容格式
2. 将整理好的 Markdown 通过 A2A 发给飞书 agent
3. 飞书 agent 调用飞书 API 创建文档
4. 完成后把文档链接推回主 agent
5. 主 agent 把链接告诉用户

**价值：** 主 agent 负责"想"，飞书 agent 负责"做"，职责清晰。

---

### 场景三：定时监控 + 主动通知

**问题：** 用户希望飞书有重要待办时，自动推送到 Telegram。

**解法：**

1. 飞书 agent 的 cron job 每天早上 9:00 运行
2. 检查到有未处理的文档审批或@提醒
3. 飞书 agent 通过 A2A 发消息给主 agent
4. 主 agent 在 Telegram 推送给用户

**价值：** 飞书事件主动桥接到用户常用渠道，无需用户手动查看。

---

## 配置方式

### 前提：多 agent 设置

确保 `openclaw.json` 中已定义多个 agent：

```json5
{
  agents: {
    list: [
      { id: "main" },
      {
        id: "xiao-niu",
        name: "xiao-niu",
        workspace: "/Users/yourname/.openclaw/workspace-xiao-niu",
        agentDir: "/Users/yourname/.openclaw/agents/xiao-niu/agent",
      },
    ],
  },
}
```

---

### 第一步：开启 Session 可见性

默认情况下，每个 agent 只能看到自己的 session。要允许跨 agent 访问，需设置：

```json5
{
  tools: {
    sessions: {
      // "self"  - 只有当前 session
      // "tree"  - 当前 session + 子 agent（默认）
      // "agent" - 当前 agent 的所有 session
      // "all"   - 所有 agent 的所有 session（A2A 必须）
      visibility: "all",
    },
  },
}
```

---

### 第二步：开启 Agent to Agent 通信

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["main", "xiao-niu"], // 填写允许互发消息的 agent id 列表
    },
  },
}
```

> ⚠️ `allow` 列表中的 agent 才能互相通信，未列出的 agent 之间仍然隔离。

---

### 第三步：重启 Gateway

```bash
openclaw gateway restart
```

---

### 完整配置示例

```json5
{
  agents: {
    list: [
      { id: "main" },
      {
        id: "xiao-niu",
        workspace: "/Users/yourname/.openclaw/workspace-xiao-niu",
        agentDir: "/Users/yourname/.openclaw/agents/xiao-niu/agent",
      },
    ],
  },
  tools: {
    sessions: {
      visibility: "all",
    },
    agentToAgent: {
      enabled: true,
      allow: ["main", "xiao-niu"],
    },
  },
}
```

---

## 让 Agent 互相认识

Agent 之间没有自动发现机制——一个 agent 不知道另一个 agent 的存在，除非被明确告知。

### 方式一：对话时临时告知（即时生效，不持久）

直接在对话中描述另一个 agent：

> "xiao-niu 是一个飞书 agent，agentId 是 `xiao-niu`，负责读写飞书文档和群聊"

agent 在当前会话中就能使用这个信息，但**新会话启动后会忘记**。

---

### 方式二：写入 MEMORY.md（推荐，长期有效）

将其他 agent 的信息写入 workspace 的 `MEMORY.md`，agent 每次启动都会读取：

```markdown
## 已知 Agent

### xiao-niu
- agentId: xiao-niu
- 渠道: 飞书
- 能力: 读写飞书文档、飞书群聊、知识库操作
- 联系方式: sessions_send("agent:xiao-niu:main", ...)
```

这样无需每次提醒，agent 自然知道对方存在、能做什么、怎么联系。

---

### 方式三：写入 AGENTS.md 或自定义配置文件

如果想让 agent 在系统层面就了解协作关系，可以在 workspace 的 `AGENTS.md` 中增加说明：

```markdown
## 协作 Agent

- **xiao-niu**（飞书助手）：负责所有飞书操作，通过 `sessions_send("agent:xiao-niu:main", ...)` 联系
```

---

### 三种方式对比

| 方式 | 持久性 | 适用场景 |
|---|---|---|
| 对话时告知 | ❌ 会话结束即忘 | 临时测试、一次性任务 |
| 写入 MEMORY.md | ✅ 长期有效 | 固定协作伙伴（推荐） |
| 写入 AGENTS.md | ✅ 长期有效 | 系统级协作关系说明 |

---

## 注意事项

- **两个配置缺一不可：** `sessions.visibility=all` 控制"能看到"，`agentToAgent.enabled=true` 控制"能发消息"
- **agent 不会自动发现彼此**，需要通过 session key 明确指定目标
