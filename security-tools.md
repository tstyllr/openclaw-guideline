# 安全-工具

## 工具默认配置

默认配置 = 全部工具开放 + 直接跑主机 + 无沙箱 + exec 不需确认

## 工具配置方式

工具（tools）可以在`~/.openclaw/openclaw.json`文件中进行配置，分三层：全局默认、Agent级、沙箱专属

三层都支持 allow / deny / profile，deny 永远最优先。

```
全局默认（tools.*）
↓ 被覆盖
Agent 级（agents.list[].tools.*）
↓ 叠加
沙箱专属（tools.sandbox.tools.* 或 agents.list[].tools.sandbox.tools.*）
```

示例配置文件：

```json
// ~/.openclaw/openclaw.json
{
  // ① 全局层：所有 agent、所有 session 都受约束
  "tools": {
    "profile": "messaging",
    "deny": ["gateway", "cron"],

    // ③ 沙箱层：只在 session 处于沙箱中时额外生效
    "sandbox": {
      "tools": {
        "allow": ["group:fs", "group:runtime"]
      }
    }
  },

  "agents": {
    "list": [
      {
        "id": "work",
        // ② Agent 层：只影响这个 agent
        "tools": {
          "profile": "full",
          "deny": ["sessions_spawn"],
          "sandbox": {
            "tools": {
              "deny": ["exec"] // 这个 agent 在沙箱里还额外禁 exec
            }
          }
        }
      }
    ]
  }
}
```

## 最佳实践

工具权限配置分三个维度，从粗到细：

───

一、tools.profile — 预设档位（最快上手）

```json
{
  "tools": {
    "profile": "messaging" // 最常用的安全基线
  }
}
```

| 档位      | 包含工具 | 适用场景                      |
| --------- | -------- | ----------------------------- |
| full      | 全部工具 | 受信任的私人主 session        |
| messaging | 仅消息类 | 半公开 bot，没有文件/命令权限 |
| minimal   | 几乎没有 | 最严格，近乎只能聊天          |

───

二、tools.deny / tools.allow — 精细黑白名单

```json
{
  "tools": {
    "profile": "messaging",
    // 在 messaging 基础上再额外禁掉这些：
    "deny": [
      "group:automation", // 自动化工具组（cron 等）
      "group:runtime", // 运行时控制（spawn/send sessions）
      "group:fs", // 文件系统访问
      "sessions_spawn", // 创建子 agent
      "sessions_send", // 跨 session 发消息
      "gateway", // 改 config/重启 gateway
      "cron" // 创建定时任务
    ]
  }
}
```

───

三、tools.exec — 命令执行权限

```json
{
  "tools": {
    "exec": {
      "security": "deny", // deny | allowlist | full
      "ask": "always", // 每次执行都要确认
      "host": "sandbox" // 在沙箱里跑，不污染主机
    }
  }
}
```

| security 值 | 行为                     |
| ----------- | ------------------------ |
| deny        | 完全禁止 exec            |
| allowlist   | 只允许 safeBins 里的命令 |
| full        | 允许任意命令（危险）     |

───

四、tools.fs — 文件系统限制

```json
{
  "tools": {
    "fs": {
      "workspaceOnly": true // 只能访问 workspace 目录，不能碰其他文件
    }
  }
}
```

───

五、tools.elevated — 高权限工具

```json
{
  "tools": {
    "elevated": {
      "enabled": false // 完全关掉 elevated 工具
    }
  }
}
```

───

六、按 Agent 单独配置（最灵活）

```json
{
  "agents": {
    "list": [
      {
        "id": "main", // 主 agent，全权限
        "tools": { "profile": "full" }
      },
      {
        "id": "public-bot", // 公开 bot，严格限制
        "tools": {
          "profile": "messaging",
          "deny": ["gateway", "cron", "sessions_spawn"],
          "exec": { "security": "deny" },
          "fs": { "workspaceOnly": true }
        }
      }
    ]
  }
}
```

───

推荐安全基线（直接可用）

```json
// ~/.openclaw/openclaw.json
{
  "tools": {
    "profile": "messaging",
    "deny": [
      "gateway",
      "cron",
      "sessions_spawn",
      "sessions_send",
      "group:automation",
      "group:runtime",
      "group:fs"
    ],
    "exec": { "security": "deny", "ask": "always" },
    "fs": { "workspaceOnly": true },
    "elevated": { "enabled": false }
  }
}
```

配完之后跑 `openclaw security audit` 验证一下，有问题它会直接告诉你怎么改

## 目前支持的tool

所有内置 Tool 清单

| Tool             | 说明                                            |
| ---------------- | ----------------------------------------------- |
| exec             | 执行 shell 命令                                 |
| bash             | 执行 bash 脚本                                  |
| process          | 管理后台进程（list/poll/log/write/kill）        |
| read             | 读取文件内容                                    |
| write            | 创建/覆盖文件                                   |
| edit             | 精确编辑文件片段                                |
| apply_patch      | 应用 diff patch                                 |
| web_search       | Brave 搜索网络                                  |
| web_fetch        | 抓取网页内容（HTML → markdown）                 |
| browser          | 控制浏览器（点击/截图/抓内容/自动化）           |
| canvas           | 控制 Canvas 画布                                |
| nodes            | 管理配对节点（摄像头/屏幕/通知/运行命令）       |
| image            | 用图像模型分析图片                              |
| pdf              | 分析 PDF 文档                                   |
| message          | 跨 channel 发消息/操作（发送/反应/搜索/线程等） |
| cron             | 管理定时任务                                    |
| gateway          | 修改 config、重启 gateway                       |
| sessions_list    | 列出所有 session                                |
| sessions_history | 查看 session 历史                               |
| sessions_send    | 向其他 session 发消息                           |
| sessions_spawn   | 创建子 agent/session                            |
| session_status   | 查看当前 session 状态                           |
| memory_search    | 语义搜索记忆文件                                |
| memory_get       | 读取记忆文件片段                                |

───

──

所有 Tool 分组（分组是语法糖，方便批量操作，本质上等价于把组里的工具逐个列出来）

| 分组             | 包含的工具                                                                     |
| ---------------- | ------------------------------------------------------------------------------ |
| group:runtime    | exec, bash, process                                                            |
| group:fs         | read, write, edit, apply_patch                                                 |
| group:sessions   | sessions_list, sessions_history, sessions_send, sessions_spawn, session_status |
| group:memory     | memory_search, memory_get                                                      |
| group:web        | web_search, web_fetch ← 之前漏掉了这个！                                       |
| group:ui         | browser, canvas                                                                |
| group:automation | cron, gateway                                                                  |
| group:messaging  | message                                                                        |
| group:nodes      | nodes                                                                          |
| group:openclaw   | 所有内置工具的总集合（不含插件）                                               |

───
