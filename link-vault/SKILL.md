---
name: link-vault
description: 链接收纳 — Use when the user shares a link from Douyin（抖音）, Bilibili（B站）, Xiaohongshu（小红书）, YouTube, X/Twitter, or any web page, and wants its content transcribed, summarized, and archived into their Obsidian vault. Routes to the right extraction tool per platform, generates a structured Markdown note with an AI summary, key points, and the full transcript or body text, writes it into the matching vault folder, and syncs to all devices via Fast Note Sync when available. 触发词：链接收纳、保存这个链接、把这个视频存到 Obsidian、总结一下这个链接。
compatibility: 需要 Obsidian 正在运行且 obsidian CLI 可用；按平台需要 agent-reach（opencli / bili-cli / yt-dlp / twitter-cli）与 douyin-mcp 服务；需要网络访问；可选 Fast Note Sync 服务用于多端同步。
metadata:
  version: "1.0.0"
  author: "SloaOVO"
---

# 链接收纳（Link Vault）

把用户发来的链接变成一篇结构化的 Obsidian 笔记：爬取 → 分析 → 预览 → 写入 → 同步。严格按步骤执行，不要跳步或合并步骤。

## 输入

- 一个链接（抖音 / B站 / 小红书 / YouTube / X / 任意网页），可附带保存意图（如「存一下」「总结这个」）。
- 用户只发链接、没有多说时，默认执行完整收纳流程。

## 前置检查（先做，缺什么先说）

1. **解析 Vault 路径**：优先用环境变量 `LINK_VAULT_PATH`；否则运行 `obsidian vaults` 取当前仓库路径。取不到就问用户，不要猜路径，更不要写死任何个人路径。
2. **按平台检查工具**（清单见 [references/platform-tools.md](references/platform-tools.md)）：抖音需要 douyin-mcp 已连接；B站 / 小红书需要 opencli 且 Chrome 已登录对应平台；YouTube 需要 yt-dlp；X 需要 twitter-cli；通用网页用 Jina Reader。
3. 任一关键工具不可用：告诉用户缺什么、怎么补，然后停下，不要带病继续。

## 工作流

### 步骤 1：解析平台

按域名判断平台与目标文件夹：

| 域名特征 | 平台 | 文件夹 |
|---|---|---|
| `douyin.com` | 抖音 | `抖音/` |
| `bilibili.com` / `b23.tv` | B站 | `B站/` |
| `xiaohongshu.com` / `xhslink.com` | 小红书 | `小红书/` |
| `youtube.com` / `youtu.be` | YouTube | `YouTube/` |
| `x.com` / `twitter.com` | X | `X/` |
| 其他 | 通用网页 | `网页/` |

识别不了平台时，按通用网页处理。

### 步骤 2：爬取内容

按 [references/platform-tools.md](references/platform-tools.md) 中对应平台的命令执行。核心原则：**视频必须拿到字幕 / 转录全文**，只有标题和描述不算完成。

- 爬取失败（网络、风控、404、无权限）：向用户报告具体原因并终止，**绝不创建空笔记或残缺笔记**。
- 视频确实无字幕：标注「本视频无字幕」，用标题 + 描述 + 元信息成文，并明确告诉用户。

### 步骤 3：生成笔记

按 [assets/note-template.md](assets/note-template.md) 的结构生成：frontmatter（platform / source / date / tags）、标题、一句话摘要、原始内容全文（字幕不截断）、3–8 条关键观点（每条一句话、一针见血）、留空的「我的思考」区。标签 = 平台标签 + 2–4 个内容关键词。

### 步骤 4：先预览，后写入

把完整的笔记预览展示给用户，确认后再写入。

- 先用 `mkdir -p` 建好目标文件夹。
- 用 bash heredoc 写文件，分隔符必须带单引号（`<< 'ENDOFFILE'`），防止正文里的 `$` 等符号被 shell 展开。
- 不要用 obsidian-cli 写长内容（YAML 分隔符、引号等特殊字符会导致解析失败）；不要用沙箱内的文件工具写 Vault 路径（可能被沙箱限制拦截）。
- 文件名：`YYYY-MM-DD <标题>.md`，标题截断到 50 字符。

### 步骤 5：完成提示

告知用户笔记已保存到 `<平台文件夹>/<文件名>.md`；Fast Note Sync 运行中会自动同步到其他设备，未运行则笔记已存在本地，服务启动后自动同步。

## 错误处理速查

| 场景 | 行为 |
|---|---|
| 链接无法识别平台 | Jina Reader 通用网页兜底 |
| 爬取失败（网络 / 风控 / 404） | 报告具体原因，终止，不创建笔记 |
| 视频无字幕 | 标注「本视频无字幕」，保存标题 + 描述 + 链接 |
| opencli 报 `BROWSER_CONNECT` | 提示用户打开 Chrome、登录对应平台、确认 OpenCLI 扩展已启用 |
| B站 `bili video` 失败 | 改用 `opencli bilibili video <BV号> -f yaml` 备用 |
| 目标文件夹不存在 | `mkdir -p` 自动创建 |
| 更深的部署 / 连接问题 | 查 [references/troubleshooting.md](references/troubleshooting.md) |

## 关键规则

- 不要跳过字幕提取：视频的核心价值在字幕，标题 + 描述远远不够。
- 先展示笔记预览再写入，让用户看到内容后再落盘。
- heredoc 分隔符一定要带单引号：`<< 'ENDOFFILE'`，不是 `<< ENDOFFILE`。
- 任何一步失败都如实报告并停下，不要用空笔记或猜测内容交差。
