# memory_search —— OpenClaw 语义记忆检索工具

## 什么是 memory_search？

`memory_search` 是 OpenClaw 内置的**语义搜索工具**，用于检索 agent workspace 里的记忆文件。它不依赖关键词精确匹配，而是通过 **embedding 向量**理解语义，找出最相关的内容片段。

---

## 检索范围

默认索引以下文件：

- `MEMORY.md` —— 长期记忆（curated，仅主会话加载）
- `memory/*.md` —— 所有子目录 Markdown 文件（按主题或日期命名的笔记）

可通过 `extraPaths` 配置扩展索引额外目录（见下文）。

---

## 和关键词搜索的区别

|          | memory_search                              | grep / 关键词搜索    |
| -------- | ------------------------------------------ | -------------------- |
| 匹配方式 | 语义理解                                   | 文字完全匹配         |
| 示例     | 搜"认证失败"也能找到"too many failed auth" | 必须输入一模一样的词 |
| 底层模型 | 本地 embedding 模型（GGUF）                | 无                   |

---

## 何时会被调用？

### 1. 系统强制触发（Mandatory Recall）

在回答以下类型的问题前，agent **必须**先执行 `memory_search`：

- 先前工作、历史决策
- 日期、时间相关的事项
- 人物、偏好、待办事项

### 2. 会话启动时

新会话开始，agent 按照 `AGENTS.md` 规范主动检索最近记忆。

### 3. Agent 自主判断

当 agent 判断问题可能在记忆文件中有相关信息时，也会主动调用。

### 4. 用户主动触发

可以直接在聊天中要求 agent 搜索：

```
"帮我搜一下记忆里关于 Feishu 的内容"
"memory_search 查一下 ACP 相关笔记"
```

---

## memory_search vs memory_get

|          | memory_search                      | memory_get               |
| -------- | ---------------------------------- | ------------------------ |
| 本质     | 语义搜索                           | 文件读取                 |
| 输入     | 自然语言查询                       | 文件路径 + 可选行范围    |
| 输出     | 多个相关片段 + 评分                | 指定文件的完整文本       |
| 适合场景 | 不知道在哪个文件，按主题找         | 已知具体文件路径和行号   |
| 触发时机 | 问题模糊、主题相关                 | 直接读取某个记忆文件     |
| 路径限制 | 无需指定路径                       | 只能读 `MEMORY.md` 或 `memory/` 内的文件 |

### 两步工作流（常见模式）

`memory_search` 找到线索后，用 `memory_get` 读取完整内容：

```
memory_search("ACP 配置")
  → 返回：memory/2026-03-06-acp-setup.md 第 5-20 行有相关内容
    → memory_get("memory/2026-03-06-acp-setup.md", from=5, lines=20)
        → 读取完整内容
```

`memory_search` 每个片段上限约 700 字符，想看完整上下文时用 `memory_get` 补读。

### 一句话区别

> **memory_search** = 图书馆检索系统（不知道书在哪，按主题找）  
> **memory_get** = 直接去书架取书（知道在哪，直接拿）

---

## 工具参数

```
query        必填，搜索问题或关键词（支持自然语言）
maxResults   可选，返回结果数量
minScore     可选，相似度阈值（0~1）
```

---

## 索引机制

### 索引存储位置

```
~/.openclaw/memory/<agentId>.sqlite
```

### 触发索引的时机

| 触发条件              | 说明                                               |
| --------------------- | -------------------------------------------------- |
| 会话启动时            | 调度一次同步                                       |
| 文件变化时            | 文件 watcher 检测到变动，防抖 **1.5 秒**后异步重建 |
| 调用 memory_search 时 | 若 index 为脏则先同步                              |
| 定时间隔              | 按配置周期定期同步                                 |

### 触发完全重建的条件

以下任意一项变化都会触发全量重建：

- embedding 模型（provider / model）
- API endpoint 指纹
- 分块参数（chunk size / overlap）

### 分块规则

- 每块约 **400 token**，重叠 **80 token**
- 返回片段上限约 **700 字符**

---

## 默认是否开启？

**默认开启**，但需要 embedding provider 才能真正工作。

### Provider 自动选择顺序

| 优先级 | Provider  | 条件                                |
| ------ | --------- | ----------------------------------- |
| 1      | `local`   | 配置了 `local.modelPath` 且文件存在 |
| 2      | `openai`  | 能找到 OpenAI API key               |
| 3      | `gemini`  | 能找到 Gemini API key               |
| 4      | `voyage`  | 能找到 Voyage API key               |
| 5      | `mistral` | 能找到 Mistral API key              |
| ❌     | 无        | 以上都没有 → memory search 保持禁用 |

---

## 配置方式

### 使用本地模型（离线，免费）

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "local",
      },
    },
  },
}
```

也可以指定模型：

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "local",
        local: {
          modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
        },
      },
    },
  },
}
```

默认本地模型约 **0.6 GB**，首次使用时自动下载。

### 使用 OpenAI

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
      },
    },
  },
}
```

### 扩展索引目录（extraPaths）

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: [
          "../team-docs", // workspace 相对路径
          "/srv/shared-notes/overview.md", // 绝对路径（单个文件）
        ],
      },
    },
  },
}
```

注意事项：

- 只索引 `.md` 文件
- 目录递归扫描
- 软链接会被忽略

### 完全关闭 memory search

```json5
{
  plugins: {
    slots: {
      memory: "none",
    },
  },
}
```

---

## 文件监视器（Watcher）

文件监视器**内置在 OpenClaw Gateway 进程**中，随 Gateway 启动/停止。

```
OpenClaw Gateway（长驻进程）
└── Memory Search 管理器
    ├── SQLite 向量索引
    ├── 文件监视器（watcher）
    │   └── 监视 MEMORY.md + memory/ 目录
    │       防抖 1.5s → 标记 dirty → 异步重建索引
    └── Embedding 模型（本地/远程）
```

| 情况                           | 结果                         |
| ------------------------------ | ---------------------------- |
| Gateway 运行中，写了新记忆文件 | 1.5s 内自动索引，无需重启    |
| Gateway 重启                   | 会话启动时重新调度同步       |
| Gateway 未运行                 | 无 watcher，文件变化不被感知 |

---

## 记忆文件命名规范

AGENTS.md 期望的日记格式为 `memory/YYYY-MM-DD.md`（每天一个文件），session 启动时自动读取今天和昨天的文件。

但也可以采用**按主题命名**的方式（如 `memory/2026-03-06-feishu-setup.md`），这些文件不会在启动时自动读取，但可以通过 `memory_search` 检索到。

---

## 相关文档

- OpenClaw 官方文档：`/concepts/memory.md`
- 配置参考：`agents.defaults.memorySearch`
