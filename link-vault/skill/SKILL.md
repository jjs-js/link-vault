---
name: link-vault
description: 链接收纳 — 发一个链接，AI自动爬取分析，存入Obsidian并同步到所有设备
---

# 链接收纳

触发方式：`/sync <链接>`

收到链接后严格按以下步骤执行，**不要跳过或合并步骤**。

## 常量

- Vault 路径：通过 `obsidian vaults` 自动获取，或手动配置
- FNS 服务：`localhost:9000`（自动检测文件变更并同步，无需手动触发 MCP）

## 步骤 1：URL 解析

检查域名确定平台和文件夹：

| 域名特征 | 平台 | 文件夹 |
|---------|------|--------|
| `douyin.com` | 抖音 | `抖音/` |
| `bilibili.com` / `b23.tv` | B站 | `B站/` |
| `xiaohongshu.com` / `xhslink.com` | 小红书 | `小红书/` |
| `youtube.com` / `youtu.be` | YouTube | `YouTube/` |
| `x.com` / `twitter.com` | X | `X/` |
| 其他 | 通用网页 | `网页/` |

## 步骤 2：内容爬取

按平台选择工具。**失败时向用户报告原因并终止，绝不创建空笔记。**

### 抖音
```
# 获取视频信息
mcp__douyin-mcp__parse_douyin_video_info(share_link="<URL>")
# 提取字幕
mcp__douyin-mcp__extract_douyin_text(share_link="<URL>")
```

### B站

从 URL 中提取 BV 号（如 `BV1eZgt6YEP5`）。

先获取视频信息：
```bash
bili video <BV号>
```

字幕提取 — **优先 opencli，无需尝试 bili audio（pipx 环境 PyAV 安装麻烦）**：
```bash
opencli bilibili subtitle <BV号>
```
- 若报 `BROWSER_CONNECT`：提示「请打开 Chrome，确保 B站已登录且 OpenCLI 扩展已启用」
- 若无字幕（返回空）：标注「本视频无字幕」，跳过字幕环节，仅用视频信息

### 小红书
```bash
opencli xiaohongshu note "<URL>" -f yaml
```

### YouTube
```bash
yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hans,zh,en" --skip-download -o "/tmp/%(id)s" "<URL>"
cat /tmp/<VIDEO_ID>.*.vtt
```

### X/Twitter
```bash
twitter tweet "<URL>"
```

### 通用网页
```bash
curl -s "https://r.jina.ai/<URL>"
```

## 步骤 3：AI 分析

基于爬取内容生成笔记。**必须在回答中展示笔记预览**，然后写入。

笔记结构：
```markdown
---
platform: <平台名>
source: <原始链接>
date: <当前日期 YYYY-MM-DD>
tags:
  - <平台标签>
  - <2-4个内容关键词>
---

# <标题>

> [!abstract] AI 摘要
> 一句话概括核心内容

## 📝 原始内容

**视频/来源信息**
- 作者/UP主：<名称>
- 时长/字数：<信息>
- 播放/互动：<数据>（如有）

<字幕/正文全文>

## 💡 关键观点

- 观点1
- 观点2
- ...

## 🤔 我的思考

<!-- 在此记录你的想法 -->
```

要求：
- 关键观点 3-8 条，每条一句话，一针见血
- 原始内容保留完整字幕/正文（视频字幕特别重要，不要截断）
- 有视频信息的（UP主、时长、播放量等）放在「视频信息」小节

## 步骤 4：写入 Obsidian

**必须用 bash heredoc，不要用 obsidian-cli（特殊字符会报错），不要用 write_file（sandbox 限制）。**

文件名：`YYYY-MM-DD <标题>.md`，标题截断至 50 字符。

```bash
mkdir -p "E:/Users/ASUS/Documents/1/<平台文件夹>"
cat > "E:/Users/ASUS/Documents/1/<平台文件夹>/<文件名>.md" << 'ENDOFFILE'
<完整笔记内容>
ENDOFFILE
```

注意：heredoc 分隔符必须用 `'ENDOFFILE'`（单引号防止变量展开），确保 Markdown 中的 `$` 等符号不被 shell 解释。

## 步骤 5：完成提示

写入成功后提示：

> ✅ 笔记已保存到 Obsidian：`<平台>/<文件名>.md`
> 
> FNS 运行中时自动同步到 Windows 和 Android 端。如 FNS 未运行，笔记已在本地，启动后自动同步。

## 错误处理速查

| 场景 | 行为 |
|------|------|
| 链接无法识别平台 | Jina Reader 通用网页兜底 |
| 爬取失败（网络/风控/404） | 报告具体原因，终止，不创建笔记 |
| 视频无字幕 | 标注「本视频无字幕」，保存标题+描述+链接 |
| opencli BROWSER_CONNECT | 提示用户打开 Chrome 并登录对应平台 |
| B站 `bili video` 失败 | 尝试用 `opencli bilibili video <BV号> -f yaml` 备用 |
| 文件夹不存在 | `mkdir -p` 自动创建 |

## 注意事项

- **不要跳过字幕提取**：视频的核心价值在字幕，标题+描述远远不够
- **heredoc 一定要用单引号**：`<< 'ENDOFFILE'` 不是 `<< ENDOFFILE`
- **不要用 write_file 工具写 vault 文件**：sandbox 限制会报错
- **不要用 obsidian-cli create 写长内容**：YAML 分隔符、引号等会导致命令解析失败
- **先展示笔记预览再写入**：让用户看到内容后再写入文件
