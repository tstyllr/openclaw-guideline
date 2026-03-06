# 斜杠命令（Slash Commands）

## 什么是斜杠命令？

斜杠命令是以 `/` 开头、直接发送给 OpenClaw Gateway 的控制指令。
它们**不经过大模型处理**，由 Gateway 直接执行并返回结果，因此：

- 响应速度快
- 不消耗 token
- 不占用上下文窗口

## 处理流程对比

**斜杠命令：**
```
用户发送 /status
    ↓
Telegram → OpenClaw Gateway
    ↓
Gateway 直接处理，返回结果（大模型不参与）
```

**普通消息：**
```
用户发送 "你好"
    ↓
Telegram → OpenClaw Gateway
    ↓
Gateway 转发给大模型
    ↓
大模型生成回复 → 用户看到
```

---

## 命令类型

### 1. 普通命令（Text Commands）

独立发送的 `/xxx` 消息，所有 channel 均支持。

### 2. 指令（Directives）

如 `/think`、`/model`、`/reasoning` 等，可以：
- **内嵌在消息中**：当作"一次性提示"，不持久化
- **单独发送**：持久化到会话设置

### 3. 内联快捷方式（Inline Shortcuts）

部分命令可以嵌入普通消息中，执行后剩余文字继续正常处理：
- `/help`、`/commands`、`/status`、`/whoami`

示例：`今天天气怎么样 /status` — 先执行 status，再回答天气问题。

---

## 常用命令速查

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息 |
| `/status` | 查看当前会话状态（token 用量、模型等）|
| `/new` 或 `/reset` | 开启新会话（清空上下文）|
| `/model` | 查看或切换 AI 模型 |
| `/model list` | 列出可用模型 |
| `/think <level>` | 设置推理深度（off/minimal/low/medium/high/xhigh）|
| `/reasoning on\|off` | 开关推理模式（推理过程单独发送）|
| `/verbose on\|off` | 开关详细输出（调试用）|
| `/usage off\|tokens\|full\|cost` | 控制 token/费用显示 |
| `/tts` | 控制语音合成 |
| `/stop` | 停止当前运行 |
| `/restart` | 重启 Gateway |
| `/whoami` 或 `/id` | 查看自己的 sender id |
| `/context` | 查看上下文组成 |
| `/subagents list` | 查看子 Agent 列表 |
| `/skill <name> [input]` | 按名称运行技能 |

### 默认关闭、需手动启用的命令

| 命令 | 配置项 | 说明 |
|------|--------|------|
| `/bash <cmd>` | `commands.bash: true` | 执行主机 shell 命令 |
| `/config` | `commands.config: true` | 读写配置文件 |
| `/debug` | `commands.debug: true` | 运行时配置覆盖（不持久化）|

---

## 跨 Channel 差异

### 功能层面：一致

所有 channel 支持相同的命令集（平台专属命令除外），**功能相同，体验不同**。

### 体验层面：不同

| 平台 | 原生命令支持 | 体验 |
|------|------------|------|
| Telegram | ✅ | 输入 `/` 自动弹出命令菜单 |
| Discord | ✅ | 支持 autocomplete、按钮菜单 |
| Slack | ⚠️ 需手动配置 | 需在 Slack 后台逐一注册 |
| 飞书 / WhatsApp / Signal / iMessage | ❌ | 纯文本输入，无提示菜单，但命令仍然有效 |

> 飞书发送 `/status` 没有弹出提示，但 OpenClaw 依然识别并执行。

### 平台专属命令（Discord）

| 命令 | 说明 |
|------|------|
| `/vc join\|leave\|status` | 语音频道控制 |
| `/focus <target>` | 将线程绑定到 session/subagent |
| `/unfocus` | 解除线程绑定 |
| `/agents` | 查看当前线程绑定的 Agent |

---

## 自定义斜杠命令

OpenClaw **不支持直接新建**自定义斜杠命令，但可以通过 **Skill（技能）** 间接实现：

- 创建一个标记为 `user-invocable` 的 Skill
- OpenClaw 会自动将其注册为斜杠命令
- 可通过 `/skill <name> [input]` 调用，或直接 `/<name>`

**相关配置项：**

```json
{
  "commands": {
    "native": "auto",        // 是否注册原生命令（auto/true/false）
    "nativeSkills": "auto",  // 是否将 Skill 注册为原生命令
    "text": true,            // 是否解析文本中的 /xxx 命令
    "bash": false,           // 是否启用 /bash 命令
    "config": false,         // 是否启用 /config 命令
    "debug": false,          // 是否启用 /debug 命令
    "allowFrom": {           // 命令授权白名单
      "*": ["user1"]
    }
  }
}
```

---

## 小结

| 特性 | 说明 |
|------|------|
| 处理方 | Gateway 直接处理，不经过大模型 |
| 速度 | 快，不消耗 token |
| 跨平台 | 功能一致，体验因平台而异 |
| 扩展方式 | 通过 Skill 注册自定义命令 |
