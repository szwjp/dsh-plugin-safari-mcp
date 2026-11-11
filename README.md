# dsh-plugin-safari-mcp

把 **Apple 官方 Safari MCP server**（Safari 27+ 内置的 `safaridriver --mcp`）通过 DSH 自带的
`@deepseek-ai/dsh-mcp-client` 桥接进 DeepSeek Harness，17 个 WebDriver 工具以
`mcp__safari__<tool>` 的原生工具名出现，和内置工具一样可被模型直接调用。

本包**不含 JavaScript 代码**，只提供一个 bundle patch：真正的 MCP 客户端、工具发现、
schema 注册、超时/取消、`tools/list_changed` 重同步都由 DSH 内置的 mcp-client 完成。

## 工具（17 个）

| 工具 | 用途 |
|---|---|
| `mcp__safari__list_tabs` / `switch_tab` / `create_tab` / `close_tab` | 标签页管理（自动化会话内） |
| `mcp__safari__navigate_to_url` / `wait_for_navigation` / `page_info` | 导航与就绪判断 |
| `mcp__safari__get_page_content` | WebKit 文本提取（推荐阅读页面；含元素 UID，可配合 `$uid(N)`） |
| `mcp__safari__evaluate_javascript` | 页内执行 JS（可进 iframe） |
| `mcp__safari__page_interactions` | 批量 DOM 交互：click / type / keyPress / scroll / selectText… |
| `mcp__safari__screenshot` | 截图 PNG，返回文件路径 |
| `mcp__safari__list_network_requests` / `get_network_request` | 网络请求与响应详情 |
| `mcp__safari__browser_console_messages` / `browser_dialogs` | console 日志 / JS 对话框 |
| `mcp__safari__set_viewport_size` / `set_emulated_media` | 视口与媒体类型模拟 |

## 前置条件（macOS）

1. Safari 27 或更新（或 Safari Technology Preview 247+）。
2. 打开远程自动化：
   - Safari → 设置 → 开发者 → 勾选「允许远程自动化与外部代理」
     （英文：Allow remote automation and external agents）
   - 或终端：`sudo safaridriver --enable`
3. 首次调用工具时，Safari 会弹出自动化授权；允许即可。

## 安装

从 GitHub 安装（推荐，公开分享用）：

```text
plugin_manager.install_bundle(target: github:szwjp/dsh-plugin-safari-mcp)
```

从本地目录安装（改代码即时生效，适合自己维护）：

```text
plugin_manager.install_bundle(target: file:/Users/<you>/.dsh/plugins/dsh-plugin-safari-mcp)
```

等价的手工方式（profile：`~/.dsh/profiles/desktop`）：

```jsonc
// package.json
"dependencies": { "dsh-plugin-safari-mcp": "github:szwjp/dsh-plugin-safari-mcp" },
"dsh": { "profile": { "bundles": [ /* … */, "dsh-plugin-safari-mcp" ] } }
```

安装后 DSH 的热应用会立即生效；冷启动时同样从 profile bundles 加载。

## 配套 Skill（建议一起装）

仓库内 `skills/safari-browser/SKILL.md` 是配套的 Agent Skill，把使用规则固化下来：

- 任何导航后**重新 `list_tabs` 取 handle**（跨站导航会换句柄）；
- 读页面**优先 `get_page_content`**（结构化 textTree + 元素 UID），而不是 `evaluate_javascript`；
- 需要**真实登录态/用户现有标签**时改用 `safari_*`（AppleScript），不要用隔离的 `mcp__safari__*`；
- `set_emulated_media` 直接跳过（Safari 27.2 上稳定超时）；
- `list_tabs` 的 `active` 字段不可信；用 `page_info` 判断当前页。

安装到 DSH 的全局 skill 目录即可被自动发现（DSH 会热刷新 catalog）：

```sh
mkdir -p ~/.dsh/skills/safari-browser
cp skills/safari-browser/SKILL.md ~/.dsh/skills/safari-browser/
```

## 与其他方案的边界

- **和 `dsh-plugin-macos` 的 `safari_*` 工具是互补关系**：那一组走 AppleScript，操作你**真实
  Safari 的现有标签与登录态**；本组走 WebDriver，是**隔离的自动化会话**（实测：看不到普通窗口
  的标签，cookie 不与日常浏览共享），适合调试、验证渲染、抓网络/console 证据。
- Safari MCP 的自动化会话启动时会跳过窗口恢复（`Skipping session restore`），不会污染你的浏览。

## 实测记录（Safari 27.2 / macOS 27.2.1，2026-11-11）

15/17 个工具端到端跑通：

| 工具 | 结果 |
|---|---|
| `list_tabs` / `create_tab` / `close_tab` / `switch_tab` | ✅ |
| `navigate_to_url` | ✅ 返回带元素 UID 的 textTree，可直接用 `page_interactions` 的 `node` 参数点 UID |
| `page_info` / `wait_for_navigation` / `get_page_content` / `evaluate_javascript` | ✅ |
| `page_interactions` | ✅ 文本定位点击、UID 点击、跨站导航均验证 |
| `screenshot` / `set_viewport_size` / `browser_console_messages` / `browser_dialogs` | ✅ |
| `set_emulated_media` | ❌ 稳定报 `Timed out awaiting the page target`（Safari 27.2 上疑似不支持，待上游确认） |
| `get_network_request` | ⚠️ 未测：`list_network_requests` 在测试页返回 0 条（网络捕获未生效，原因待查） |

两个使用时必须知道的坑：

1. **tab handle 跨站导航后会变**：`page-B8ACE3DF…` 点链接跳到 rfc-editor.org 后句柄变成
   `page-D75CA03A…`，用旧句柄 `close_tab` 会报 `Could not find browsing context`。
   **规则：任何导航动作之后重新 `list_tabs` 拿新句柄，不要缓存句柄。**
2. `list_tabs` 里所有 tab 的 `active` 可能都是 `false`（即便 `switch_tab` 已生效，
   `page_info` 已指向新页）——以 `page_info` 为准，`active` 字段不可靠。

## 故障排查

| 现象 | 处理 |
|---|---|
| 工具完全没出现 | 看 DSH 启动日志里 `mcp-client(safari)` 的报错；确认 `safaridriver --mcp` 能启动 |
| 首次调用报 `You must enable 'Allow remote automation'…` | 按上面「前置条件 2」打开开关 |
| `close_tab` 报 `Could not find browsing context` | 句柄因导航失效，先 `list_tabs` 取新句柄 |
| 某次调用超时 | 冷启动/慢页面；调大 `toolCallTimeoutMs`（默认 120s） |
| 需要卸载 | 从 profile `package.json` 删除依赖与 bundles 项，或 `plugin_manager` 禁用后移除 |

## 许可

MIT。`@deepseek-ai/dsh-mcp-client` 与 `safaridriver` 均随 DSH / macOS 提供，本包仅做配置桥接。
