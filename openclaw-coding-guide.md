# OpenClaw Coding 功能指南

> 整理自 2026-03-06 与龙虾老师的对话

---

## 一、OpenClaw 内置 Coding 能力概览

OpenClaw 在编程方面提供四个层次的能力：

### 1. 🛠️ exec 工具（直接执行 Shell）

可以直接在沙箱里运行 shell 命令，适合简单任务：

- 支持 PTY（伪终端，适合 TUI 类 CLI）
- 支持后台执行（`background: true`）
- 支持超时控制（`timeout` 参数）
- 支持工作目录指定（`workdir`）

**适用场景：** 简单脚本、git 操作、文件处理、编译构建等。

---

### 2. 🤖 ACP（Agent Client Protocol）接入外部 Coding 代理

通过 ACP 协议，将编程任务委托给专业的 coding harness 工具。

**目前支持的代理：**

| 代理 ID    | 工具                  |
| ---------- | --------------------- |
| `codex`    | OpenAI Codex CLI      |
| `claude`   | Anthropic Claude Code |
| `gemini`   | Google Gemini CLI     |
| `opencode` | OpenCode              |
| `pi`       | Pi                    |
| `kimi`     | Kimi                  |

**使用示例：**

```
/acp spawn codex --mode persistent
/acp spawn claude --mode oneshot
```

---

### 3. 🦾 Coding Agent Skill（内置技能）

OpenClaw 内置 `coding-agent` skill，可将复杂编码任务委派给 Codex / Claude Code 等，以后台进程方式运行。

**适用场景：**

- 构建 / 创建新功能
- 重构大型代码库
- Review PR（在临时目录中 spawn）
- 需要文件探索的迭代式编码

**不适合：** 简单单行修改（直接用 edit 工具）、仅读代码（用 read 工具）。

---

### 4. 🔀 Sub-agents（子代理）

Spawn 隔离的子代理会话来处理编码任务，支持并行工作或长时间运行的任务。

---

## 二、ACP 环境配置步骤

> **默认状态：** ACP 功能默认未启用，`acpx` 插件状态为 `disabled`，需手动配置。

### 第一步：启用 acpx 插件

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

### 第二步：配置 ACP 基础参数

```bash
openclaw config set acp.enabled true
openclaw config set acp.backend acpx
openclaw config set acp.defaultAgent codex
```

### 第三步：设置权限模式（重要！）

ACP 以非交互式方式运行，必须提前配置权限策略，否则写文件 / 执行命令时会报错：

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

**`permissionMode` 可选值：**

| 值              | 行为                               |
| --------------- | ---------------------------------- |
| `approve-all`   | 自动批准所有文件写入和 shell 命令  |
| `approve-reads` | 仅自动批准读取；写入和执行需要确认 |
| `deny-all`      | 拒绝所有权限请求                   |

**`nonInteractivePermissions` 可选值：**

| 值     | 行为                                         |
| ------ | -------------------------------------------- |
| `fail` | 遇到需要交互的权限提示时，中止并报错（默认） |
| `deny` | 静默拒绝权限并继续执行（优雅降级）           |

### 第四步：配置允许的代理列表（可选）

```bash
openclaw config set acp.allowedAgents '["codex","claude","gemini"]'
```

### 第五步：重启 Gateway

```bash
openclaw gateway restart
```

### 第六步：验证配置

```bash
/acp doctor
```

---

## 三、使用 ACP 的前提条件

要使用具体的代理，需在本机安装对应的 CLI 工具并配置 API Key：

| 代理        | 安装命令                                   | 所需 Key          |
| ----------- | ------------------------------------------ | ----------------- |
| Codex       | `npm install -g @openai/codex`             | OpenAI API Key    |
| Claude Code | `npm install -g @anthropic-ai/claude-code` | Anthropic API Key |
| Gemini CLI  | `npm install -g @google/gemini-cli`        | Google API Key    |

---

## 四、常用 ACP 命令速查

| 命令                 | 说明                                        |
| -------------------- | ------------------------------------------- |
| `/acp spawn codex`   | 启动 Codex ACP 会话                         |
| `/acp status`        | 查看当前 ACP 会话状态                       |
| `/acp steer <指令>`  | 向运行中的会话发送引导指令                  |
| `/acp cancel`        | 取消当前会话的运行中任务                    |
| `/acp close`         | 关闭会话并解除线程绑定                      |
| `/acp doctor`        | 检查 ACP 后端健康状态，输出可操作的修复建议 |
| `/acp sessions`      | 列出近期 ACP 会话                           |
| `/acp model <model>` | 为会话设置模型覆盖                          |

---

## 五、ACP vs Sub-agents 对比

| 维度       | ACP 会话                            | Sub-agent                 |
| ---------- | ----------------------------------- | ------------------------- |
| 运行时     | ACP 后端插件（acpx）                | OpenClaw 原生子代理运行时 |
| 主要命令   | `/acp ...`                          | `/subagents ...`          |
| Spawn 方式 | `sessions_spawn` + `runtime:"acp"`  | `sessions_spawn`（默认）  |
| 适合场景   | 接入外部 coding harness（Codex 等） | OpenClaw 原生任务委派     |

---

_文档生成时间：2026-03-06 | 来源：OpenClaw 龙虾老师教学对话_
