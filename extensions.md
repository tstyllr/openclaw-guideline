# Extensions（插件）

## 概述

在 OpenClaw 中，**Plugin = Extension**，两个词可以互换使用。

Extension 是用 TypeScript 编写的代码模块，用于给 OpenClaw 扩展额外功能。当你需要 OpenClaw 核心之外的能力时（或者想把可选功能与主安装分离），就用 Extension 来实现。

## Extension 能做什么

一个 Extension 可以注册以下内容：

- **消息渠道** — 新的聊天平台（Telegram、Discord、飞书、Matrix……）
- **Agent 工具** — 让 AI 能调用的新能力
- **CLI 命令** — 新的 `openclaw xxx` 子命令
- **斜杠命令** — 不经过 AI 直接执行的命令（如 `/mystatus`）
- **后台服务** — 持续运行的进程
- **Skills** — Agent 技能包
- **模型 OAuth 认证流程** — 让用户在 OpenClaw 内完成第三方登录

## 两个 Extension 目录

OpenClaw 有两个主要的 extension 目录，性质完全不同：

### 内置目录（Stock）

```
/path/to/openclaw/node_modules/openclaw/extensions/
```

- 随 OpenClaw 一起打包发布的**官方插件**
- 包含约 40 个插件：telegram、discord、feishu、voice-call、memory-core 等
- **不要修改这里的文件**——升级 OpenClaw 会覆盖
- 这些是 OpenClaw 的"标准库"

### 用户目录（Global）

```
~/.openclaw/extensions/
```

- 你自己安装或开发的插件放这里
- 升级 OpenClaw **不会影响**这个目录
- 通过 `openclaw plugins install` 安装的包也会放到这里

类比：前者像系统自带 App，后者像你自己装的 App。

## 加载优先级

OpenClaw 按以下顺序扫描 extension，**先找到的优先**：

1. `plugins.load.paths`（config 里手动指定的路径）
2. `<workspace>/.openclaw/extensions/`（workspace 级）
3. `~/.openclaw/extensions/`（全局用户级）
4. 内置 extensions（随 OpenClaw 打包，**默认禁用**）

> ⚠️ 注意：内置 extensions 虽然排在最后，但它们在 `plugins list` 中优先显示为 `stock` 来源。如果用户目录和内置目录存在同名插件（相同 id），会触发 duplicate 警告，先加载的（stock）会覆盖后加载的（global）。

## 常用命令

```bash
# 查看所有插件状态
openclaw plugins list

# 从 npm 安装官方插件
openclaw plugins install @openclaw/voice-call

# 安装本地插件（复制到 ~/.openclaw/extensions/）
openclaw plugins install ./my-plugin

# 启用 / 禁用
openclaw plugins enable <id>
openclaw plugins disable <id>

# 更新（仅限 npm 安装的插件）
openclaw plugins update <id>
openclaw plugins update --all

# 查看插件详情
openclaw plugins info <id>

# 检查插件健康状态
openclaw plugins doctor
```

> 修改插件配置后，需要重启 Gateway 才能生效：`openclaw gateway restart`

## 官方可用插件（节选）

| 插件             | npm 包                 | 说明                     |
| ---------------- | ---------------------- | ------------------------ |
| Telegram         | 内置                   | Telegram 渠道            |
| Feishu           | 内置                   | 飞书/Lark 渠道           |
| Memory (Core)    | 内置                   | 文件记忆搜索（默认启用） |
| Memory (LanceDB) | 内置                   | 向量长期记忆（默认禁用） |
| Voice Call       | `@openclaw/voice-call` | 语音通话（Twilio）       |
| Microsoft Teams  | `@openclaw/msteams`    | Teams 渠道               |
| Matrix           | `@openclaw/matrix`     | Matrix 渠道              |
| Discord          | 内置                   | Discord 渠道             |
| Signal           | `@openclaw/signal`     | Signal 渠道              |

## Plugin 配置示例

在 `openclaw.json` 中：

```json
{
  "plugins": {
    "enabled": true,
    "allow": ["voice-call"],
    "entries": {
      "voice-call": {
        "enabled": true,
        "config": { "provider": "twilio" }
      }
    }
  }
}
```

字段说明：

- `enabled` — 全局开关（默认 true）
- `allow` — 白名单（建议填写以消除安全警告）
- `deny` — 黑名单（deny 优先于 allow）
- `load.paths` — 额外的插件路径
- `entries.<id>` — 单个插件的开关和配置

## 清理冲突的 Extension

如果出现 `duplicate plugin id` 警告（用户目录和内置目录有同名插件），清理步骤：

```bash
# 1. 删除用户目录中的冲突插件
rm -rf ~/.openclaw/extensions/<plugin-name>

# 2. 清除 config 中的安装记录
openclaw config unset plugins.installs.<plugin-id>

# 3. 重启 Gateway
openclaw gateway restart
```

## 安全注意事项

- Extension 和 Gateway **运行在同一进程**，拥有完全信任的权限
- 只安装你信任的插件
- 建议在 `plugins.allow` 中明确列出允许加载的插件 id
- `openclaw plugins install` 使用 `npm install --ignore-scripts` 安装依赖，不执行 lifecycle 脚本
- 插件代码不得通过 symlink 或路径穿越逃出插件目录

## 开发自己的 Extension

一个最简单的 Extension 结构：

```
my-plugin/
├── openclaw.plugin.json   # 插件清单（必须）
├── index.ts               # 入口文件
└── package.json
```

入口文件导出格式：

```typescript
// 函数形式
export default function (api) {
  api.registerTool({ name: "my_tool", ... });
}

// 对象形式
export default {
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerTool({ name: "my_tool", ... });
  }
};
```

更多开发细节参考官方文档：[Plugin agent tools](/plugins/agent-tools)
