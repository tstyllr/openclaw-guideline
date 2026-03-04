# 安全-沙盒

## 沙盒有什么用

沙盒的作用：

- 沙盒可以让工具在Docker容器中运行
- 隔离不同session的记忆

备注：

- Gateway还是运行在主机上，只是在需要执行工具的时候在Docker容器中执行

## 默认配置

默认关闭沙盒功能

## 启动方式

───

🔒 沙盒机制三个维度

1. 触发模式（Mode）

- "off" → 不沙盒，工具直接跑在宿主机
- "non-main" → 只沙盒非主 session（群聊、频道等）
- "all" → 所有 session 都跑在沙盒里

2. 容器范围（Scope）

- "session" → 每个 session 一个容器（默认）
- "agent" → 每个 agent 共享一个容器
- "shared" → 所有沙盒 session 共用一个容器

3. Workspace 访问权限

- "none" → 沙盒看不到你的 workspace（默认最安全）
- "ro" → 只读挂载 workspace
- "rw" → 读写挂载 workspace

───

🧰 配置示例

最常见的启用方式：

```json
// ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "scope": "session",
        "workspaceAccess": "none"
      }
    }
  }
}
```

多 agent 场景可以差异化：主 agent 无沙盒，公开/家庭 agent 强制沙盒 + 限制工具。

───

⚠️ 重要说明

• 沙盒默认是关闭的，需要手动配置开启
• Gateway 进程本身始终在宿主机，沙盒只隔离工具执行
• elevated 模式的 exec 会绕过沙盒跑在宿主机
• 沙盒容器默认没有网络，装包需要额外配置

───

## 实操流程

───

完整流程：为 xiaoshu 开启沙盒

───

第一步：构建沙盒 Docker 镜像

镜像的 Dockerfile 在 GitHub 源码里，需要自己构建。先创建一个临时文件：

创建 Dockerfile：

```bash
cat > /tmp/Dockerfile.sandbox << 'EOF'
FROM debian:bookworm-slim@sha256:98f4b71de414932439ac6ac690d7060df1f27161073c5036a7553723881bffbe

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
 bash \
 ca-certificates \
 curl \
 git \
 jq \
 python3 \
 ripgrep \
 && rm -rf /var/lib/apt/lists/\*

RUN useradd --create-home --shell /bin/bash sandbox
USER sandbox
WORKDIR /home/sandbox

CMD ["sleep", "infinity"]
EOF

```

构建镜像（名字必须是这个）：

```bash
docker build -t openclaw-sandbox:bookworm-slim -f /tmp/Dockerfile.sandbox /tmp
```

构建完验证一下：

```bash
docker images openclaw-sandbox
```

───

第二步：修改 xiaoshu agent 的配置

```json
// ~/.openclaw/openclaw.json
{
  "agents": {
    "list": [
      {
        "id": "xiaoshu",
        "name": "xiaoshu",
        "workspace": "/Users/yangzhicong/.openclaw/workspace-xiaoshu",
        "agentDir": "/Users/yangzhicong/.openclaw/agents/xiaoshu/agent",
        "sandbox": {
          "mode": "all",
          "scope": "session",
          "workspaceAccess": "none"
        }
      }
    ]
  }
}
```

验证配置写进去了：

```bash
openclaw config get agents
```

───

第三步：重启 Gateway

```bash
openclaw gateway restart
```

───

第四步：验证沙盒是否生效

```bash
openclaw sandbox explain --agent xiaoshu
```

看到 mode: all + runtime: docker 就说明沙盒已经开启。

也可以查看容器状态（有 session 触发后才会出现）：

```bash
openclaw sandbox list
```

───

## 自动销毁容器

───

默认销毁规则（源码确认）

```js
DEFAULT_SANDBOX_IDLE_HOURS = 24; // 闲置超24小时 → 销毁
DEFAULT_SANDBOX_MAX_AGE_DAYS = 7; // 存在超7天 → 销毁
```

prune 逻辑每 5 分钟检查一次（HOT_CONTAINER_WINDOW_MS = 300s），满足任一条件就会 docker rm -f 容器。

───

如何禁止自动销毁？

将两者都设为 0 即可完全禁用：

```json
// ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "prune": {
          "idleHours": 0, // 0 = 禁用空闲清理
          "maxAgeDays": 0 // 0 = 禁用最大年龄清理
        }
      }
    }
  }
}
```

也可以针对特定 agent 单独设置（注意：scope=shared 时 per-agent prune 配置会被忽略，统一用全局配置）：

```json
// ~/.openclaw/openclaw.json
{
  "agents": {
    "list": [
      {
        "id": "my-agent",
        "sandbox": {
          "prune": { "idleHours": 0, "maxAgeDays": 0 }
        }
      }
    ]
  }
}
```

───

## 启用网络

───

开启沙盒网络

```json
// ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "docker": {
          "network": "bridge" // 开启网络
        }
      }
    }
  }
}
```

───

可用的网络模式

| 值               | 效果                             |
| ---------------- | -------------------------------- |
| "none"（默认）   | 完全断网                         |
| "bridge"         | 标准 Docker 桥接网络，可访问外网 |
| "host"           | ❌ 被封锁，不允许                |
| "container:<id>" | ❌ 默认封锁（命名空间劫持风险）  |

───

特殊情况：host 和 container 为啥被封？

• network: "host" 直接共享主机网络栈，等于绕过隔离，OpenClaw 硬拒。
• network: "container:<id>" 可以加入另一个容器的网络命名空间，存在绕过风险，默认也封了。如果非要用，需要开启破戒开关：

```json
{
  "sandbox": {
    "docker": {
      "dangerouslyAllowContainerNamespaceJoin": true
    }
  }
}
```

───

建议

开网络之前想清楚目的——沙盒断网的设计本身就是为了防止 agent 偷偷往外发数据。开了 bridge 之后，agent 可以访问任意外网，安全边界就弱了一截。如果只需要访问特定服务，考虑用自定义 Docker 网络配合防火墙规则限流量。

## browser工具

沙盒里可以用 browser 工具！🦞 OpenClaw 专门为沙盒提供了一个独立的浏览器容器。

───

沙盒浏览器是独立容器

```json
{
  "sandbox": {
    "browser": {
      // 默认自动启动，browser 工具需要时会自动拉起
      "autoStart": true,
      "autoStartTimeoutMs": 30000,

      // 浏览器容器用独立的 Docker 网络（不是全局 bridge）
      "network": "openclaw-sandbox-browser"
    }
  }
}
```

沙盒浏览器镜像需要单独构建：

[scripts/sandbox-browser-setup.sh](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-browser-setup.sh)

───

访问控制

| 配置项                                    | 作用                                     |
| ----------------------------------------- | ---------------------------------------- |
| cdpSourceRange                            | 限制 CDP 入口的 CIDR，比如 172.21.0.1/32 |
| allowHostControl                          | 允许沙盒 session 控制主机浏览器          |
| allowedControlUrls                        | 限制可控制的 URL 白名单                  |
| allowedControlHosts / allowedControlPorts | 限制可控制的主机/端口                    |

───

noVNC 观察

沙盒浏览器还支持 noVNC 远程观察，默认密码保护，OpenClaw 会生成短期 token URL 供访问。

───

简单说

| 场景                     | browser 工具可用？            |
| ------------------------ | ----------------------------- |
| 无沙盒                   | ✅ 用主机浏览器               |
| 有沙盒（未配置 browser） | ✅ 自动启动沙盒浏览器容器     |
| 需要用主机浏览器         | 需开启 allowHostControl: true |

所以沙盒不会让 browser 失效，只是浏览器也跑在容器里，和主机浏览器隔离开了。

## 开启sandbox的agent和没开启的区别

Sandbox 开启 vs 关闭对比 🦞

文件系统

|                  | 无沙盒          | 有沙盒（none）      | 有沙盒（rw）         |
| ---------------- | --------------- | ------------------- | -------------------- |
| 读写 workspace   | ✅ 直接操作主机 | ❌ 只能操作沙盒副本 | ✅ 挂载到 /workspace |
| 记忆持久化       | ✅              | ❌                  | ✅                   |
| 访问主机其他目录 | ✅ 无限制       | ❌                  | ❌（除非手动 binds） |

───

网络

|          | 无沙盒   | 有沙盒                |
| -------- | -------- | --------------------- |
| 默认网络 | 主机网络 | none（完全断网）      |
| 可配置   | —        | 可开启，但不能用 host |

───

进程隔离

|                | 无沙盒       | 有沙盒        |
| -------------- | ------------ | ------------- |
| exec 执行环境  | 主机进程空间 | Docker 容器内 |
| 能影响主机进程 | ✅           | ❌            |
| 崩溃影响       | 可能波及主机 | 仅影响容器    |

───

记忆与配置

|                          | 无沙盒             | 有沙盒（none/ro）                          |
| ------------------------ | ------------------ | ------------------------------------------ |
| MEMORY.md 写入持久       | ✅                 | ❌                                         |
| SOUL.md/USER.md 可被篡改 | ✅（有风险）       | ❌（天然保护）                             |
| 系统 prompt 注入内容     | 来自主机 workspace | 来自主机 workspace（只读注入，写操作隔离） |

───

保护配置文件不被篡改：

| 文件                  | none 模式       | rw 模式         |
| --------------------- | --------------- | --------------- |
| SOUL.md               | ❌ agent 改不了 | ✅ agent 可以改 |
| USER.md / IDENTITY.md | ❌ agent 改不了 | ✅ agent 可以改 |
| MEMORY.md             | ❌ 改了不持久   | ✅ 持久写回     |
| memory/\*.md          | ❌ 改了不持久   | ✅ 持久写回     |

───

适用场景

| 场景                 | 推荐配置                       |
| -------------------- | ------------------------------ |
| 主会话、个人助手     | 无沙盒，或 mode: "non-main"    |
| 执行不可信代码/任务  | 沙盒 + workspaceAccess: "none" |
| 多用户共享 agent     | 沙盒 + scope: "session"        |
| 需要记忆的沙盒 agent | 沙盒 + workspaceAccess: "rw"   |

───

最佳实践建议

个人主 agent（你信任它）→ rw 没问题，记忆持久化很重要。

处理不可信输入的 agent（比如帮陌生人处理文件）→ 考虑用 none，接受记忆不持久的代价，换取配置安全。

───
