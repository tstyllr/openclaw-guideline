# OpenClaw Browser 工具指南

> 本文档基于实际使用体验整理，涵盖 browser 工具的技术原理、Profile 模式、使用场景与注意事项。

---

## 一、技术原理

### 底层架构

Browser 工具基于 **Playwright**（微软出品的浏览器自动化框架），通过 **Chrome DevTools Protocol（CDP）** 与浏览器通信：

```
OpenClaw → Playwright API → Chrome DevTools Protocol (CDP) → 浏览器进程
```

CDP 是 Chrome 暴露的底层控制协议，可以精确控制页面导航、DOM 操作、网络请求等。

### 不依赖视觉，依赖结构

Browser 工具**不是**通过截图分析像素来操作浏览器的，而是直接读取页面的**可访问性树（Accessibility Tree）**：

```
snapshot → 获取语义结构（button/textbox/table...）→ 通过 ref 操作元素
```

因此，有无图形界面对 browser 工具的核心功能**没有任何影响**。截图（`screenshot`）只是可选的视觉辅助，不是必须的。

---

## 二、支持的浏览器

Browser 工具通过 CDP 协议控制浏览器，**只支持 Chromium 内核**的浏览器：

| 浏览器             | 支持 |
| ------------------ | ---- |
| Google Chrome      | ✅   |
| Brave              | ✅   |
| Microsoft Edge     | ✅   |
| Chromium（开源版） | ✅   |
| Chrome Canary      | ✅   |
| Firefox            | ❌   |
| Safari             | ❌   |

### 自动检测顺序

未指定 `executablePath` 时，OpenClaw 按以下顺序查找浏览器：

1. 系统默认浏览器（若为 Chromium 系）
2. Chrome
3. Brave
4. Edge
5. Chromium
6. Chrome Canary

---

## 三、核心功能

### 1. 快照（Snapshot）

```
browser(action="snapshot")
```

抓取当前页面的可访问性树，返回结构化 UI 元素（非原始 HTML），每个元素有唯一 `ref`（如 `e12`）供后续操作引用。

### 2. 动作（Act）

```
browser(action="act", request={kind:"click", ref:"e12"})
```

支持的操作类型：`click`、`type`、`fill`、`press`、`hover`、`drag`、`select`、`evaluate`（执行 JS）

### 3. 截图

```
browser(action="screenshot")
```

返回页面截图，用于视觉确认，非必须。

### 4. 导航

```
browser(action="open", url="https://...")
browser(action="navigate", url="https://...")
```

---

## 四、Profile 模式

OpenClaw 提供两种内置 Profile，适用于不同场景：

### `openclaw` Profile（沙盒模式）

- OpenClaw **主动启动**的独立浏览器实例
- 用户数据目录：`~/.openclaw/browser/openclaw/user-data`
- 初始状态为空白，**无任何 Cookie 或登录信息**
- 登录后 Cookie **持久保存**，下次启动自动恢复登录态
- 无需任何扩展，开箱即用

### `chrome` Profile（接管模式）

- **被动接管**用户正在使用的 Chrome
- 通过 **Browser Relay 扩展**（MV3）attach 到指定 Tab
- 完全复用用户现有的 Cookie 和登录状态
- 只控制用户手动 attach 的 Tab

### 对比总览

| 特性                | `openclaw`     | `chrome`         |
| ------------------- | -------------- | ---------------- |
| 浏览器实例          | 独立新实例     | 用户现有 Chrome  |
| 初始登录态          | 空白           | 继承现有登录     |
| 需要扩展            | ❌             | ✅ Browser Relay |
| 控制范围            | 完整（多 Tab） | 仅 attach 的 Tab |
| 是否影响用户 Chrome | 完全隔离       | 共享进程         |
| Headless 支持       | ✅             | ❌               |

### 如何选择

```
需要操作已登录的网站（且不想重新登录）？
├─ 有图形界面 → chrome profile（最省事）
└─ 无图形界面 / 想隔离 → openclaw profile（首次登录一次，之后持久）

操作公开网站，不需要登录？
└─ openclaw profile（干净、安全）
```

---

## 五、Headless 模式（无图形界面）

### 完全支持

Headless Chrome 支持与有界面 Chrome **几乎完全相同**的功能，包括：

- JS 执行、CSS 渲染、动态内容
- 网络请求、Cookie、Session
- DOM 操作（snapshot/act）
- 截图、PDF 生成
- 多 Tab 管理

### 配置方法

在 `~/.openclaw/openclaw.json` 中配置：

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### Headless 的限制

| 功能         | 说明                                                      |
| ------------ | --------------------------------------------------------- |
| Chrome 扩展  | ❌ 不支持（因此 `chrome` profile 无法在 Headless 下使用） |
| GPU 加速     | 默认禁用，WebGL 性能稍弱                                  |
| 音视频播放   | 受限                                                      |
| 某些反爬检测 | 部分网站能检测 Headless 特征                              |

### Headless 下的 Profile 选择

```
有图形界面：openclaw ✅  chrome ✅
无图形界面：openclaw ✅  chrome ❌
```

Headless 环境下**只能使用 `openclaw` profile**，`chrome` 扩展模式依赖真实窗口，无法在服务器上运行。

---

## 六、Linux 安装注意事项

Ubuntu 的 `apt install chromium` 安装的是 **snap 版**，受 AppArmor 沙箱限制，OpenClaw 无法启动。

**推荐安装官方 Chrome `.deb` 包：**

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y
```

---

## 七、实际应用场景

| 场景             | 推荐 Profile          | 说明                         |
| ---------------- | --------------------- | ---------------------------- |
| 操作内网 ERP/CRM | `chrome`              | 复用已登录状态，无需重新登录 |
| 公开网站数据抓取 | `openclaw`            | 干净环境，无副作用           |
| 服务器定时自动化 | `openclaw` + Headless | 无图形界面，稳定运行         |
| 网页自动化测试   | `openclaw`            | 隔离环境，结果可复现         |
| 多账号隔离操作   | 自定义多个 Profile    | 各自独立 Cookie              |

### 创建自定义 Profile

```bash
# 为不同业务系统创建独立 Profile
openclaw browser create-profile --name erp-system
openclaw browser create-profile --name crm-system

# 指定特定浏览器
openclaw browser create-profile \
  --name brave \
  --executable-path "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
```

---

## 八、使用技巧

1. **ref 要保持一致**：snapshot 后拿到的 ref，后续 act 要传同一个 `targetId`，避免 ref 失效
2. **优先用 snapshot 而非 screenshot**：snapshot 是结构化数据，更适合 AI 理解和操作
3. **不要滥用 `act:wait`**：只在没有其他可靠页面状态判断方式时才使用
4. **登录态持久化**：`openclaw` profile 的 Cookie 会保存，无需每次重新登录
5. **安全意识**：`chrome` profile 激活后 AI 可完全操控该 Tab，用完及时 detach

---

_文档生成时间：2026-03-06_
