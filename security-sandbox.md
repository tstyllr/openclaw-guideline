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
openclaw sandbox explain
```

看到 mode: all + runtime: docker 就说明沙盒已经开启。

也可以查看容器状态（有 session 触发后才会出现）：

```bash
openclaw sandbox list
```

───
